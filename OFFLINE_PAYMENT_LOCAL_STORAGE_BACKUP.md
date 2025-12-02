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
    expired: number;
  }> {
    const storage = await this.getStorage();
    const now = Date.now();
    
    return {
      total: storage.payments.length,
      pending: storage.payments.filter(p => p.status === 'pending').length,
      syncing: storage.payments.filter(p => p.status === 'syncing').length,
      synced: storage.payments.filter(p => p.status === 'synced').length,
      failed: storage.payments.filter(p => p.status === 'failed').length,
      expired: storage.payments.filter(p => now > p.expiresAt && p.status !== 'synced').length,
    };
  }
}
```

---

## Phase 2: Capture Offline Payments

### 2.1 Update Payment Collection Hook

**File:** `gb2-terminal-expo/hooks/useStripePayment.js`

Add storage after successful payment collection:

```javascript
// After successful payment confirmation (around line 200)
if (confirmResult.paymentIntent) {
    await logger.info(`✅ [Native] Payment confirmed successfully`, {
        paymentIntentId: confirmResult.paymentIntent.id,
        status: confirmResult.paymentIntent.status
    }, executionContext);
    
    // Store offline payment in local storage for backup
    if (offlineBehavior === 'force_offline') {
        const offlinePayment: OfflinePaymentRecord = {
            id: uuid.v4(),
            paymentIntentId: confirmResult.paymentIntent.id,
            amount: amountInCents,
            currency: 'usd',
            status: 'pending',
            createdAt: Date.now(),
            collectedAt: Date.now(),
            expiresAt: Date.now() + (24 * 60 * 60 * 1000), // 24 hours
            paymentMethod: {
                type: confirmResult.paymentIntent.paymentMethod?.type,
                cardBrand: confirmResult.paymentIntent.paymentMethod?.cardPresent?.brand,
                last4: confirmResult.paymentIntent.paymentMethod?.cardPresent?.last4,
            },
            metadata: payload.paymentIntentMetaData,
            syncAttempts: 0,
        };
        
        await OfflinePaymentStorageService.addPayment(offlinePayment);
    }
    
    // ... rest of code
}
```

---

## Phase 3: Sync to Backend

### 3.1 Update Stripe Callbacks

**File:** `gb2-terminal-expo/hooks/useStripeCallbacks.js`

```javascript
onDidForwardPaymentIntent: async (paymentIntent) => {
    await logger.info("✅ Payment forwarded to Stripe successfully", {
        paymentIntentId: paymentIntent.id,
        amount: paymentIntent.amount,
        status: paymentIntent.status
    });
    
    // Update local storage: mark as synced to Stripe
    await OfflinePaymentStorageService.updatePaymentStatus(
        paymentIntent.id,
        'syncing',
        { syncedToStripeAt: Date.now() }
    );
    
    // Now sync to backend
    try {
        await syncPaymentToBackend(paymentIntent);
        
        // Mark as fully synced
        await OfflinePaymentStorageService.updatePaymentStatus(
            paymentIntent.id,
            'synced',
            { syncedToBackendAt: Date.now() }
        );
        
        // Remove from storage after 24 hours (keep for audit trail)
        setTimeout(async () => {
            await OfflinePaymentStorageService.removePayment(paymentIntent.id);
        }, 24 * 60 * 60 * 1000);
        
    } catch (error) {
        await logger.error("Failed to sync payment to backend", { error, paymentIntentId: paymentIntent.id });
        await OfflinePaymentStorageService.updatePaymentStatus(
            paymentIntent.id,
            'failed',
            {
                lastError: {
                    code: error.code || 'BACKEND_SYNC_FAILED',
                    message: error.message,
                    timestamp: Date.now()
                },
                syncAttempts: (await OfflinePaymentStorageService.getAllPayments())
                    .find(p => p.paymentIntentId === paymentIntent.id)?.syncAttempts + 1 || 1
            }
        );
    }
    
    postWebMessage(null, "goodbricks.paymentForwardedSuccess", paymentIntent);
},

onDidFailForwardingPaymentIntent: async (error) => {
    await logger.error("❌ Payment forwarding to Stripe FAILED", {
        paymentIntentId: error.paymentIntentId,
        errorCode: error.code,
        errorMessage: error.message
    });
    
    // Update local storage with error
    await OfflinePaymentStorageService.updatePaymentStatus(
        error.paymentIntentId,
        'failed',
        {
            lastError: {
                code: error.code,
                message: error.message,
                timestamp: Date.now()
            },
            lastSyncAttempt: Date.now(),
            syncAttempts: (await OfflinePaymentStorageService.getAllPayments())
                .find(p => p.paymentIntentId === error.paymentIntentId)?.syncAttempts + 1 || 1
        }
    );
    
    postWebMessage(null, "goodbricks.paymentForwardedFailed", {
        paymentIntentId: error.paymentIntentId,
        error: error,
        actionDisplayName: "Offline Payment Sync Failed"
    });
},
```

---


