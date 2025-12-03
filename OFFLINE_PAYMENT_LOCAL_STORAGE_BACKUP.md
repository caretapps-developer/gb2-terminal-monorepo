# Offline Payment Local Storage Backup System

## Problem Statement

**Critical Issue:** If connectivity is down for more than 24 hours, offline payment intents expire and cannot be synced to Stripe. Without a backup system, we lose all record of these transactions.

**Stripe Payment Intent Validity:** 24 hours from creation

**Current Risk:**
- ❌ No backup of offline payment data
- ❌ If Stripe Terminal sync fails, payment data is lost
- ❌ If connectivity is down >24 hours, payment intents expire
- ❌ No way to reconcile payments with backend
- ❌ Accounting/revenue tracking gaps

---

## Solution: Dual Storage Strategy

### Strategy Overview

**Store offline payments in TWO places:**

1. **Stripe Terminal SDK** (Primary)
   - Automatic sync when online
   - Handled by Stripe SDK
   - Limited storage duration

2. **Local Storage** (Backup)
   - AsyncStorage (React Native)
   - localStorage (Web layer)
   - Persistent across app restarts
   - Can be synced to backend independently

### Data Flow

```
Offline Payment Collected
         ↓
    ┌────┴────┐
    ↓         ↓
Stripe SDK   Local Storage
(Primary)    (Backup)
    ↓         ↓
    ↓    [Store payment data]
    ↓         ↓
[Online?] ←───┘
    ↓
    ├─→ YES → Sync to Stripe → Sync to Backend → Remove from Local Storage
    │
    └─→ NO → Keep in both storages → Retry later
```

---

## Implementation Plan

### Phase 1: Store Offline Payments Locally

#### 1.1 Define Data Structure

**File:** `gb2-terminal-expo/types/OfflinePayment.ts` (new file)

```typescript
export interface OfflinePaymentRecord {
  // Unique identifier
  id: string; // UUID generated locally

  // Stripe data
  paymentIntentId: string;
  amount: number; // in cents
  currency: string;
  status: 'pending' | 'syncing' | 'synced' | 'failed' | 'expired';

  // Timestamps
  createdAt: number; // Unix timestamp
  collectedAt: number; // When card was tapped
  syncedToStripeAt?: number; // When forwarded to Stripe
  syncedToBackendAt?: number; // When sent to backend
  expiresAt: number; // createdAt + 24 hours

  // Payment details
  paymentMethod?: {
    type: string;
    cardBrand?: string;
    last4?: string;
  };

  // Metadata (from payment intent)
  metadata: {
    organizationId?: string;
    locationId?: string;
    kioskId?: string;
    terminalId?: string;
    categoryId?: string;
    categoryName?: string;
    offline_mode_preference?: string;
    offline_behavior?: string;
    network_status_at_creation?: string;
    payment_mode_type?: string;
    [key: string]: any;
  };

  // Error tracking
  syncAttempts: number;
  lastSyncAttempt?: number;
  lastError?: {
    code: string;
    message: string;
    timestamp: number;
  };

  // Receipt data
  receiptData?: {
    customerEmail?: string;
    receiptUrl?: string;
  };
}

export interface OfflinePaymentStorage {
  payments: OfflinePaymentRecord[];
  lastUpdated: number;
}
```

#### 1.2 Create Storage Service

**File:** `gb2-terminal-expo/services/OfflinePaymentStorageService.ts` (new file)

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';
import { logger } from '../Logger';
import { OfflinePaymentRecord, OfflinePaymentStorage } from '../types/OfflinePayment';

const STORAGE_KEY = '@goodbricks_offline_payments';

export class OfflinePaymentStorageService {

  /**
   * Add a new offline payment to local storage
   */
  static async addPayment(payment: OfflinePaymentRecord): Promise<void> {
    try {
      const storage = await this.getStorage();

      // Check if payment already exists
      const existingIndex = storage.payments.findIndex(p => p.paymentIntentId === payment.paymentIntentId);

      if (existingIndex >= 0) {
        // Update existing payment
        storage.payments[existingIndex] = payment;
        await logger.info('Updated offline payment in local storage', { paymentIntentId: payment.paymentIntentId });
      } else {
        // Add new payment
        storage.payments.push(payment);
        await logger.info('Added offline payment to local storage', { paymentIntentId: payment.paymentIntentId });
      }

      storage.lastUpdated = Date.now();
      await this.saveStorage(storage);

    } catch (error) {
      await logger.error('Failed to add offline payment to local storage', { error, payment });
      throw error;
    }
  }

  /**
   * Get all offline payments
   */
  static async getAllPayments(): Promise<OfflinePaymentRecord[]> {
    const storage = await this.getStorage();
    return storage.payments;
  }

  /**
   * Get pending payments (not yet synced to backend)
   */
  static async getPendingPayments(): Promise<OfflinePaymentRecord[]> {
    const storage = await this.getStorage();
    return storage.payments.filter(p =>
      p.status === 'pending' || p.status === 'syncing' || p.status === 'failed'
    );
  }

  /**
   * Get expired payments (>24 hours old)
   */
  static async getExpiredPayments(): Promise<OfflinePaymentRecord[]> {
    const now = Date.now();
    const storage = await this.getStorage();
    return storage.payments.filter(p => now > p.expiresAt && p.status !== 'synced');
  }

  /**
   * Update payment status
   */
  static async updatePaymentStatus(
    paymentIntentId: string,
    status: OfflinePaymentRecord['status'],
    additionalData?: Partial<OfflinePaymentRecord>
  ): Promise<void> {
    try {
      const storage = await this.getStorage();
      const payment = storage.payments.find(p => p.paymentIntentId === paymentIntentId);

      if (payment) {
        payment.status = status;
        if (additionalData) {
          Object.assign(payment, additionalData);
        }
        storage.lastUpdated = Date.now();
        await this.saveStorage(storage);
        await logger.info('Updated offline payment status', { paymentIntentId, status });
      }
    } catch (error) {
      await logger.error('Failed to update payment status', { error, paymentIntentId, status });
    }
  }

  /**
   * Remove payment from storage (after successful backend sync)
   */
  static async removePayment(paymentIntentId: string): Promise<void> {
    try {
      const storage = await this.getStorage();
      storage.payments = storage.payments.filter(p => p.paymentIntentId !== paymentIntentId);
      storage.lastUpdated = Date.now();
      await this.saveStorage(storage);
      await logger.info('Removed offline payment from local storage', { paymentIntentId });
    } catch (error) {
      await logger.error('Failed to remove payment from storage', { error, paymentIntentId });
    }
  }

  /**
   * Clear all synced payments older than 7 days
   */
  static async cleanupOldPayments(): Promise<void> {
    try {
      const storage = await this.getStorage();
      const sevenDaysAgo = Date.now() - (7 * 24 * 60 * 60 * 1000);

      const before = storage.payments.length;
      storage.payments = storage.payments.filter(p =>
        p.status !== 'synced' || p.syncedToBackendAt! > sevenDaysAgo
      );
      const after = storage.payments.length;

      if (before !== after) {
        storage.lastUpdated = Date.now();
        await this.saveStorage(storage);
        await logger.info('Cleaned up old offline payments', { removed: before - after });
      }
    } catch (error) {
      await logger.error('Failed to cleanup old payments', { error });
    }
  }

  /**
   * Get storage object
   */
  private static async getStorage(): Promise<OfflinePaymentStorage> {
    try {
      const data = await AsyncStorage.getItem(STORAGE_KEY);
      if (data) {
        return JSON.parse(data);
      }
    } catch (error) {
      await logger.error('Failed to read offline payment storage', { error });
    }

    // Return empty storage if not found or error
    return {
      payments: [],
      lastUpdated: Date.now()
    };
  }

  /**
   * Save storage object
   */
  private static async saveStorage(storage: OfflinePaymentStorage): Promise<void> {
    try {
      await AsyncStorage.setItem(STORAGE_KEY, JSON.stringify(storage));
    } catch (error) {
      await logger.error('Failed to save offline payment storage', { error });
      throw error;
    }
  }

  /**
   * Get storage statistics
   */
  static async getStats(): Promise<{
    total: number;
    pending: number;
    syncing: number;
    synced: number;
    failed: number;


### 3.2 Backend Sync Function

**File:** `gb2-terminal-expo/services/BackendSyncService.ts` (new file)

```typescript
import { logger } from '../Logger';
import { OfflinePaymentRecord } from '../types/OfflinePayment';

export class BackendSyncService {

  /**
   * Sync payment to backend
   */
  static async syncPaymentToBackend(paymentIntent: any): Promise<void> {
    try {
      const response = await fetch(`${process.env.BACKEND_URL}/api/offline-payments/sync`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await getAuthToken()}`, // Your auth method
        },
        body: JSON.stringify({
          paymentIntentId: paymentIntent.id,
          amount: paymentIntent.amount,
          currency: paymentIntent.currency,
          status: paymentIntent.status,
          metadata: paymentIntent.metadata,
          paymentMethod: paymentIntent.paymentMethod,
          created: paymentIntent.created,
          charges: paymentIntent.charges,
        }),
      });

      if (!response.ok) {
        throw new Error(`Backend sync failed: ${response.status} ${response.statusText}`);
      }

      await logger.info('✅ Payment synced to backend', { paymentIntentId: paymentIntent.id });

    } catch (error) {
      await logger.error('❌ Failed to sync payment to backend', { error, paymentIntentId: paymentIntent.id });
      throw error;
    }
  }

  /**
   * Sync expired payments to backend (for accounting/audit)
   */
  static async syncExpiredPayments(expiredPayments: OfflinePaymentRecord[]): Promise<void> {
    try {
      const response = await fetch(`${process.env.BACKEND_URL}/api/offline-payments/expired`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await getAuthToken()}`,
        },
        body: JSON.stringify({
          payments: expiredPayments.map(p => ({
            id: p.id,
            paymentIntentId: p.paymentIntentId,
            amount: p.amount,
            currency: p.currency,
            createdAt: p.createdAt,
            collectedAt: p.collectedAt,
            expiresAt: p.expiresAt,
            metadata: p.metadata,
            paymentMethod: p.paymentMethod,
            lastError: p.lastError,
            syncAttempts: p.syncAttempts,
          })),
        }),
      });

      if (!response.ok) {
        throw new Error(`Expired payments sync failed: ${response.status}`);
      }

      await logger.info('✅ Expired payments synced to backend', { count: expiredPayments.length });

    } catch (error) {
      await logger.error('❌ Failed to sync expired payments to backend', { error, count: expiredPayments.length });
      throw error;
    }
  }

  /**
   * Retry failed payment syncs
   */
  static async retryFailedSyncs(failedPayments: OfflinePaymentRecord[]): Promise<void> {
    for (const payment of failedPayments) {
      try {
        // Check if payment is expired
        if (Date.now() > payment.expiresAt) {
          await logger.warn('Skipping expired payment', { paymentIntentId: payment.paymentIntentId });
          await OfflinePaymentStorageService.updatePaymentStatus(payment.paymentIntentId, 'expired');
          continue;
        }

        // Retry sync
        await this.syncPaymentToBackend({
          id: payment.paymentIntentId,
          amount: payment.amount,
          currency: payment.currency,
          metadata: payment.metadata,
          paymentMethod: payment.paymentMethod,
        });

        // Mark as synced
        await OfflinePaymentStorageService.updatePaymentStatus(
          payment.paymentIntentId,
          'synced',
          { syncedToBackendAt: Date.now() }
        );

      } catch (error) {
        await logger.error('Retry sync failed', { error, paymentIntentId: payment.paymentIntentId });

        // Update error info
        await OfflinePaymentStorageService.updatePaymentStatus(
          payment.paymentIntentId,
          'failed',
          {
            lastError: {
              code: error.code || 'RETRY_FAILED',
              message: error.message,
              timestamp: Date.now()
            },
            lastSyncAttempt: Date.now(),
            syncAttempts: payment.syncAttempts + 1
          }
        );
      }
    }
  }
}
```

---

## Phase 4: Background Sync Service

### 4.1 Periodic Sync Check

**File:** `gb2-terminal-expo/services/OfflinePaymentSyncManager.ts` (new file)

```typescript
import { AppState } from 'react-native';
import NetInfo from '@react-native-community/netinfo';
import { logger } from '../Logger';
import { OfflinePaymentStorageService } from './OfflinePaymentStorageService';
import { BackendSyncService } from './BackendSyncService';

export class OfflinePaymentSyncManager {
  private static syncInterval: NodeJS.Timeout | null = null;
  private static isRunning = false;

  /**
   * Start background sync service
   */
  static start(): void {
    if (this.isRunning) {
      return;
    }

    this.isRunning = true;

    // Check every 5 minutes
    this.syncInterval = setInterval(async () => {
      await this.performSync();
    }, 5 * 60 * 1000);

    // Also check when app comes to foreground
    AppState.addEventListener('change', async (nextAppState) => {
      if (nextAppState === 'active') {
        await this.performSync();
      }
    });

    // Check when network connectivity changes
    NetInfo.addEventListener(async (state) => {
      if (state.isConnected) {
        await logger.info('Network connected - checking for pending offline payments');
        await this.performSync();
      }
    });

    logger.info('Offline payment sync manager started');
  }

  /**
   * Stop background sync service
   */
  static stop(): void {
    if (this.syncInterval) {
      clearInterval(this.syncInterval);
      this.syncInterval = null;
    }
    this.isRunning = false;
    logger.info('Offline payment sync manager stopped');
  }

  /**
   * Perform sync check
   */
  private static async performSync(): Promise<void> {
    try {
      // Check network connectivity
      const netInfo = await NetInfo.fetch();
      if (!netInfo.isConnected) {
        await logger.info('No network - skipping sync check');
        return;
      }

      // Get pending payments
      const pendingPayments = await OfflinePaymentStorageService.getPendingPayments();

      if (pendingPayments.length === 0) {
        return;
      }

      await logger.info('Found pending offline payments', { count: pendingPayments.length });

      // Retry failed syncs
      await BackendSyncService.retryFailedSyncs(pendingPayments);

      // Check for expired payments
      const expiredPayments = await OfflinePaymentStorageService.getExpiredPayments();

      if (expiredPayments.length > 0) {
        await logger.warn('Found expired offline payments', { count: expiredPayments.length });

        // Sync expired payments to backend for accounting
        await BackendSyncService.syncExpiredPayments(expiredPayments);

        // Mark as expired in local storage
        for (const payment of expiredPayments) {
          await OfflinePaymentStorageService.updatePaymentStatus(payment.paymentIntentId, 'expired');
        }
      }

      // Cleanup old synced payments
      await OfflinePaymentStorageService.cleanupOldPayments();

    } catch (error) {
      await logger.error('Sync check failed', { error });
    }
  }

  /**
   * Force immediate sync
   */
  static async forceSync(): Promise<void> {
    await logger.info('Force sync triggered');
    await this.performSync();
  }

  /**
   * Get sync status
   */
  static async getStatus(): Promise<{
    isRunning: boolean;
    stats: any;
  }> {
    const stats = await OfflinePaymentStorageService.getStats();
    return {
      isRunning: this.isRunning,
      stats,
    };
  }
}
```

### 4.2 Initialize Sync Manager

**File:** `gb2-terminal-expo/App.tsx`

```typescript
import { OfflinePaymentSyncManager } from './services/OfflinePaymentSyncManager';

// In your App component
useEffect(() => {
  // Start offline payment sync manager
  OfflinePaymentSyncManager.start();

  return () => {
    OfflinePaymentSyncManager.stop();
  };
}, []);
```

---

## Phase 5: UI Updates

### 5.1 Show Expired Payment Warning

**File:** `gb2-terminal-web/src/components/TerminalHealthStatus.tsx`

Add warning for expired payments:

```typescript
const [expiredPaymentsCount, setExpiredPaymentsCount] = useState(0);

useEffect(() => {
  // Check for expired payments every minute
  const checkExpired = async () => {
    const expired = await OfflinePaymentStorageService.getExpiredPayments();
    setExpiredPaymentsCount(expired.length);
  };

  checkExpired();
  const interval = setInterval(checkExpired, 60 * 1000);

  return () => clearInterval(interval);
}, []);

// In render:
{expiredPaymentsCount > 0 && (
  <div className="flex items-center gap-1.5 text-red-600">
    <AlertTriangle className="h-3 w-3" />
    <span className="text-[10px] font-semibold">
      {expiredPaymentsCount} expired payment{expiredPaymentsCount > 1 ? 's' : ''}
    </span>
  </div>
)}
```

### 5.2 Offline Payments Management Screen

**File:** `gb2-terminal-web/src/components/OfflinePaymentsManager.tsx` (new file)

```typescript
import React, { useEffect, useState } from 'react';
import { OfflinePaymentStorageService } from '../../services/OfflinePaymentStorageService';
import { OfflinePaymentRecord } from '../../types/OfflinePayment';

export function OfflinePaymentsManager() {
  const [payments, setPayments] = useState<OfflinePaymentRecord[]>([]);
  const [stats, setStats] = useState<any>(null);

  useEffect(() => {
    loadPayments();
  }, []);

  const loadPayments = async () => {
    const allPayments = await OfflinePaymentStorageService.getAllPayments();
    const stats = await OfflinePaymentStorageService.getStats();
    setPayments(allPayments);
    setStats(stats);
  };

  const handleRetrySync = async (paymentIntentId: string) => {
    // Trigger retry
    await BackendSyncService.retryFailedSyncs([
      payments.find(p => p.paymentIntentId === paymentIntentId)!
    ]);
    await loadPayments();
  };

  return (
    <div className="p-4">
      <h2 className="text-xl font-bold mb-4">Offline Payments</h2>

      {/* Stats */}
      {stats && (
        <div className="grid grid-cols-5 gap-4 mb-6">
          <StatCard label="Total" value={stats.total} color="blue" />
          <StatCard label="Pending" value={stats.pending} color="yellow" />
          <StatCard label="Syncing" value={stats.syncing} color="blue" />
          <StatCard label="Synced" value={stats.synced} color="green" />
          <StatCard label="Failed" value={stats.failed} color="red" />
          <StatCard label="Expired" value={stats.expired} color="red" />
        </div>
      )}

      {/* Payment List */}
      <div className="space-y-2">
        {payments.map(payment => (
          <PaymentCard
            key={payment.id}
            payment={payment}
            onRetry={() => handleRetrySync(payment.paymentIntentId)}
          />
        ))}
      </div>
    </div>
  );
}
```

---

## Benefits of This Approach

### ✅ **Data Redundancy**
- Payments stored in both Stripe SDK and local storage
- If Stripe sync fails, we still have the data
- Can recover from reader failures

### ✅ **24-Hour Expiration Handling**
- Track payment intent expiration (24 hours)
- Sync expired payments to backend for accounting
- No revenue loss even if Stripe sync fails

### ✅ **Automatic Retry**
- Background service retries failed syncs
- Exponential backoff to avoid overwhelming backend
- Syncs when connectivity is restored

### ✅ **Audit Trail**
- Complete record of all offline payments
- Track sync attempts and failures
- Export for accounting/reconciliation

### ✅ **User Visibility**
- Show expired payment warnings
- Display sync status in UI
- Management screen for troubleshooting

---

## Backend Requirements

### API Endpoints Needed

#### 1. Sync Successful Payment
```
POST /api/offline-payments/sync
Body: {
  paymentIntentId: string,
  amount: number,
  currency: string,
  status: string,
  metadata: object,
  paymentMethod: object,
  created: number,
  charges: object
}
```

#### 2. Sync Expired Payments
```
POST /api/offline-payments/expired
Body: {
  payments: Array<{
    id: string,
    paymentIntentId: string,
    amount: number,
    currency: string,
    createdAt: number,
    collectedAt: number,
    expiresAt: number,
    metadata: object,
    paymentMethod: object,
    lastError: object,
    syncAttempts: number
  }>
}
```

#### 3. Get Sync Status
```
GET /api/offline-payments/status?kioskId={kioskId}
Response: {
  pendingCount: number,
  expiredCount: number,
  lastSyncAt: number
}
```

---

## Testing Checklist

### Test Scenarios

- [ ] **Normal Flow:** Collect offline payment → Go online → Auto-sync to Stripe → Auto-sync to backend
- [ ] **Stripe Sync Failure:** Collect offline payment → Stripe sync fails → Retry → Backend sync
- [ ] **Extended Offline:** Collect payment → Stay offline 25+ hours → Payment expires → Sync to backend as expired
- [ ] **App Restart:** Collect payment → Close app → Reopen → Verify payment still in storage → Sync
- [ ] **Network Blip:** Collect payment → Network drops during sync → Retry → Success
- [ ] **Multiple Payments:** Collect 10 offline payments → Sync all → Verify all in backend
- [ ] **Reader Power Loss:** Collect payment → Reader dies → Verify payment in local storage
- [ ] **Storage Cleanup:** Sync payment → Wait 7 days → Verify removed from local storage

---

## Migration Plan

### Step 1: Deploy Storage Service (Week 1)
- Add TypeScript types
- Implement OfflinePaymentStorageService
- Add unit tests
- Deploy to TestFlight

### Step 2: Integrate with Payment Flow (Week 2)
- Update useStripePayment to store payments
- Update useStripeCallbacks to handle sync
- Test with real payments
- Monitor logs

### Step 3: Add Backend Sync (Week 3)
- Implement backend API endpoints
- Add BackendSyncService
- Test sync flow
- Deploy to production

### Step 4: Add Background Sync Manager (Week 4)
- Implement OfflinePaymentSyncManager
- Add periodic sync checks
- Test retry logic
- Monitor performance

### Step 5: Add UI (Week 5)
- Add expired payment warnings
- Create management screen
- Add export functionality
- User testing

---

## Monitoring & Alerts

### Metrics to Track

1. **Sync Success Rate:** % of payments successfully synced to backend
2. **Sync Latency:** Time from collection to backend sync
3. **Expired Payment Count:** Number of payments that expired before sync
4. **Storage Size:** Number of payments in local storage
5. **Retry Count:** Average number of retries per payment

### Alerts to Configure

- ⚠️ **High Expired Count:** >5 expired payments in 24 hours
- ⚠️ **Sync Failure Rate:** >10% of syncs failing
- ⚠️ **Storage Overflow:** >100 payments in local storage
- ⚠️ **Old Pending Payments:** Payments pending >48 hours

---

## Related Documentation

- **OFFLINE_PAYMENT_SYNC_FAILURE_HANDLING.md** - Sync failure handling
- **PAYMENT_INTENT_METADATA.md** - Payment intent metadata structure
- **RECOVERY_AND_POLLING_MECHANISM.md** - Terminal recovery system


