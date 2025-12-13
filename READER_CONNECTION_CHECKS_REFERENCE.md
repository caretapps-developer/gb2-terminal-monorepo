# Reader & Connection Health Checks Reference

## Overview
This document provides a comprehensive reference for all reader and connection properties validated by the `ReaderHealthManager` component, including how they're evaluated, when they're set/unset, and what recovery actions they trigger.

---

## 📊 Health Check Properties

### 1️⃣ **`readerConnectionStatus`**

| Property | `readerConnectionStatus` |
|----------|--------------------------|
| **Type** | `string` |
| **Source** | Zustand store → Set by native Stripe SDK callback |
| **Possible Values** | `"connected"` \| `"connecting"` \| `"notConnected"` \| `"unknown"` (initial) |
| **Healthy State** | `"connected"` |
| **Unhealthy State** | Anything except `"connected"` |
| **How It's Evaluated** | `isConnected = readerConnectionStatus === "connected"` |
| **When Set** | Native SDK callback `onDidChangeConnectionStatus` → Sends `goodbricks.readerConnectionStatus` |
| **When Unset** | Reader disconnects, connection fails, or app starts |
| **Recovery Triggered** | `reader-disconnected` (if not connected) |

**Native Flow:**
```javascript
// hooks/useStripeCallbacks.js line 88-92
onDidChangeConnectionStatus: async (connectionStatus) => {
    await logger.info("onDidChangeConnectionStatus", connectionStatus)
    setReaderConnectStatus(connectionStatus)
    postWebMessage(null, "goodbricks.readerConnectionStatus", connectionStatus)
}
```

**Web Flow:**
```typescript
// ReactNativeBridge.tsx line 242-245
case "goodbricks.readerConnectionStatus":
    useGoodbricksTerminalStore.setState({ readerConnectionStatus: webViewEvent.message });
    LoggingService.info("Reader Connection Status", webViewEvent.message);
```

---

### 2️⃣ **`readerPaymentStatus`** (via `terminalData`)

| Property | `readerPaymentStatus` |
|----------|----------------------|
| **Type** | `string` |
| **Source** | Zustand store `terminalData.readerPaymentStatus` → Set by native Stripe SDK callback |
| **Possible Values** | `"ready"` \| `"waitingForInput"` \| `"processing"` \| `"readerInput"` \| `"notReady"` \| `"unknown"` (initial) |
| **Healthy State** | `"ready"` OR `"waitingForInput"` |
| **Unhealthy State** | Any other value (especially `"notReady"`) |
| **How It's Evaluated** | `isReady = readerPaymentStatus === "ready" \|\| readerPaymentStatus === "waitingForInput"` |
| **When Set** | Native SDK callback `onDidChangePaymentStatus` → Merged into `terminalData` |
| **When Unset** | Payment completes, reader disconnects, or error occurs |
| **Recovery Triggered** | `reader-not-ready` (standard layouts) or `tap-to-pay-not-waiting` (tap-to-pay layout) |

**Native Flow:**
```javascript
// hooks/useStripeCallbacks.js line 83-87
onDidChangePaymentStatus: async (paymentStatus) => {
    await logger.info("onDidChangePaymentStatus: ", paymentStatus)
    setReaderPaymentStatus(paymentStatus)
    postWebMessage(null, "goodbricks.changePaymentStatus", paymentStatus)
}

// TerminalApp.js - Merged into terminalData
const fetchTerminalData = () => {
    return {
        internetConnectionStatus: netInfoState,
        isGuidedAccessEnabled,
        readerConnectStatus,
        readerReconnectConnectStatus,
        batteryLevel,
        readerPaymentStatus,  // ← Included here
        readerOfflineStatus,
        locationInfo,
        readerSoftwareUpdateProgress
    }
}
```

---

### 3️⃣ **`connectedReader.status`**

| Property | `connectedReader.status` |
|----------|--------------------------|
| **Type** | `"online"` \| `"offline"` |
| **Source** | Zustand store `connectedReader` (full Reader object) → Set by native `getConnectedReader()` |
| **Possible Values** | `"online"` \| `"offline"` |
| **Healthy State** | `"online"` |
| **Unhealthy State** | `"offline"` |
| **How It's Evaluated** | `isReaderOnline = connectedReader?.status === "online"` |
| **When Set** | - Reader connects: `goodbricks.readerConnected` or `goodbricks.connectedReader`<br>- Health check polls: `getConnectedReader` every 30s |
| **When Unset** | Reader disconnects: `goodbricks.readerDisconnected` or `goodbricks.disconnect` |
| **Recovery Triggered** | `reader-offline` (if SDK also offline and offline mode disabled) |

**Reader Object Structure:**
```typescript
interface Reader {
  id: string;
  serialNumber: string;              // e.g., "STRM26146035736"
  deviceType: "chipper2X" | "stripeM2";
  deviceSoftwareVersion: string;
  batteryLevel: number;              // 0-1
  batteryStatus: "nominal" | "low" | "critical";
  isCharging: boolean;
  status: "online" | "offline";      // ← This is what we check!
  location: Location;
  locationId: string;
  locationStatus: "set" | "notSet";
  ipAddress: string | null;
  label: string | null;
  availableUpdate: Update | null;
  simulated: boolean;
}
```

---

### 4️⃣ **`changeOfflineStatus.sdk.networkStatus`**

| Property | `changeOfflineStatus.sdk.networkStatus` |
|----------|----------------------------------------|
| **Type** | `string` |
| **Source** | Zustand store `changeOfflineStatus` → Set by native Stripe SDK callback |
| **Possible Values** | `"online"` \| `"offline"` |
| **Healthy State** | `"online"` (unless offline mode is enabled) |
| **Unhealthy State** | `"offline"` (without offline mode enabled) |
| **How It's Evaluated** | `sdkOnline = changeOfflineStatus?.sdk?.networkStatus === "online"` |
| **When Set** | Native SDK callback `onDidChangeOfflineStatus` → Sends `goodbricks.changeOfflineStatus` |
| **When Unset** | Network disconnects, SDK loses connection |
| **Recovery Triggered** | `sdk-offline` (if offline mode not enabled) |

**Offline Status Object Structure:**
```typescript
interface OfflineStatus {
  sdk: {
    networkStatus: "online" | "offline";           // ← This is what we check!
    offlinePaymentsCount: number;                   // Number of pending offline payments
    offlinePaymentAmountsByCurrency: {              // Amounts by currency
      [currency: string]: number;                   // e.g., { "usd": 1500 }
    };
  };
}
```

---

### 5️⃣ **`offlineModeEnabled`** (Derived)

| Property | `offlineModeEnabled` |
|----------|---------------------|
| **Type** | `boolean` (derived) |
| **Source** | Calculated from `offlineModePreference` + `changeOfflineStatus` + `readerType` |
| **Possible Values** | `true` \| `false` |
| **How It's Evaluated** | See calculation below |
| **When Set** | Recalculated on every health check |
| **Recovery Impact** | Determines if SDK/reader offline is acceptable |

**Calculation Logic:**
```typescript
// ReaderHealthManager.tsx line 36-45
const getOfflineModeStatus = (offlineModePreference, changeOfflineStatus, readerType) => {
  const isTapToPayReader = readerType === "tapToPay";
  const stripeOfflineModeEnabled = changeOfflineStatus?.sdk?.offlinePaymentsCount !== undefined;
  const offlineModeEnabled = !isTapToPayReader && (
    offlineModePreference === "enabled" ||
    (offlineModePreference === "auto" && stripeOfflineModeEnabled)
  );
  // ...
};
```

**Offline Mode Logic:**
- **Tap to Pay readers:** Offline mode is NEVER enabled (not supported)
- **Bluetooth readers:**
  - `offlineModePreference === "enabled"` → Force ON
  - `offlineModePreference === "disabled"` → Force OFF
  - `offlineModePreference === "auto"` → Use Stripe Dashboard config

---

### 6️⃣ **`isInPaymentSession` (Payment Session Flag)**

| Property | `isInPaymentSession` |
|----------|---------------------|
| **Type** | Boolean |
| **Source** | Zustand store → Set by `usePaymentSession` hook in each screen component |
| **Possible Values** | `true` (payment session active), `false` (initialization/setup) |
| **Healthy State** | `true` (terminal ready for payments) |
| **Unhealthy State** | `false` (terminal in setup/initialization) |
| **How It's Evaluated** | `if (!isInPaymentSession) return;` |
| **When Set to `true`** | When entering payment screens (ReadyToPayScreen, SelectCategoryScreen, etc.) |
| **When Set to `false`** | When entering initialization screens (OrganizationSelectScreen, ConnectReaderScreen, etc.) |
| **Recovery Impact** | **Health check ONLY runs when `isInPaymentSession = true`!** |

**Payment Screens (isInPaymentSession = true):**
- `showSelectCategoryAndAmountScreen` (Single Page Layout)
- `showSelectCategoryScreen` (Multi Page Layout)
- `showEnterAmountScreen` (Multi Page Layout)
- `showReadyToPayScreen` (Tap-to-Pay Layout)
- `showMinimalReadyScreen` (Minimal Layout)
- `showPaymentProcessingScreen` (Payment Flow)
- `showCaptureCustomerInfoScreen` (Payment Flow)
- `showPaymentConfirmationScreen` (Payment Flow)
- `showQuickSuccessScreen` (Payment Flow)
- `showMinimalSuccessScreen` (Payment Flow)
- `showKeyInScreen` (Payment Flow)

**Initialization Screens (isInPaymentSession = false):**
- `initializeTerminalScreen`
- `showOrganizationSelectScreen`
- `showConnectReaderScreen`
- `showArrangeCategoriesScreen`
- `showTerminalConfigurationScreen`
- `initializeLayoutToLoad`

---

### 7️⃣ **`isGuidedAccessEnabled`** (Informational Only)

| Property | `isGuidedAccessEnabled` |
|----------|------------------------|
| **Type** | `boolean` \| `"true"` \| `"false"` \| `"unknown"` |
| **Source** | Zustand store → Set by native iOS Guided Access check |
| **Possible Values** | `true` \| `false` \| `"true"` \| `"false"` \| `"unknown"` (initial) |
| **When Set** | - App load: `initGuidedAccessCheck()` checks iOS Guided Access status<br>- Runtime: iOS event `guidedAccessStatusDidChange` |
| **When Unset** | Guided Access disabled in iOS settings |
| **Recovery Impact** | **Logged for debugging only - does NOT block health checks** |

---

### 8️⃣ **`filteredCategories`** (Informational Only)

| Property | `filteredCategories` |
|----------|---------------------|
| **Type** | `Array<Category>` |
| **Source** | Zustand store → Set during terminal configuration |
| **Possible Values** | `[]` (empty) or array of category objects |
| **When Set** | User selects categories during terminal setup |
| **When Unset** | Terminal reset or initial state |
| **Recovery Impact** | **Logged for debugging only - does NOT block health checks** |

---

### 9️⃣ **`lastDisconnectReason`** (Security Reboot)

| Property | `lastDisconnectReason` |
|----------|------------------------|
| **Type** | `string` \| `null` |
| **Source** | Zustand store → Set by native Stripe SDK `onDidDisconnect` callback |
| **Possible Values** | `"securityReboot"` \| `"disconnectRequested"` \| `"unknown"` \| `null` (initial) |
| **Healthy State** | `null` or `"disconnectRequested"` (intentional disconnect) |
| **Unhealthy State** | `"securityReboot"` or `"unknown"` (unexpected disconnect) |
| **How It's Evaluated** | `if (lastDisconnectReason === "securityReboot")` → Special 2-minute wait |
| **When Set** | Native SDK callback `onDidDisconnect(reason)` → Sends `goodbricks.disconnect` with reason |
| **When Unset** | After 2-minute security reboot wait completes → Set to `null` |
| **Recovery Impact** | **Blocks all health checks for 2 minutes!** |

**Security Reboot Timeline:**
```
T=0s    Reader disconnects
        ├─ onDidDisconnect("securityReboot") fires
        ├─ lastDisconnectReason = "securityReboot"
        ├─ lastDisconnectTime = Date.now()
        ├─ connectedReader = null
        └─ isHandlingSecurityReboot = true

T=0-120s Health checks SKIPPED
        └─ Log: "Waiting for security reboot to complete (Xs remaining)"

T=120s  Wait complete
        ├─ isHandlingSecurityReboot = false
        ├─ lastDisconnectReason = null
        ├─ recoveryState reset
        └─ Normal health checks resume

T=120s+ Standard recovery begins
        ├─ Health check detects reader disconnected
        ├─ Exponential backoff: 30s → 1m → 2m → 5m
        └─ Attempt reconnection (discover → connect)
```

**Why 2 Minutes?**
- Stripe M2 readers can take variable time to complete security reboot
- Stripe documentation does not specify exact reboot duration
- 2-minute wait reduces premature reconnection attempts that would fail
- Occurs approximately every 13 hours in production (based on observation)

---

### 9️⃣ **`readerSoftwareUpdateProgress`**

| Property | `readerSoftwareUpdateProgress` |
|----------|-------------------------------|
| **Type** | `string` |
| **Source** | Zustand store `terminalData.readerSoftwareUpdateProgress` |
| **Possible Values** | `""` (empty) or progress string |
| **Healthy State** | `""` (no update in progress) |
| **Unhealthy State** | Non-empty string (update in progress) |
| **How It's Evaluated** | `if (readerSoftwareUpdateProgress && readerSoftwareUpdateProgress !== "")` |
| **When Set** | Reader software update starts |
| **When Unset** | Update completes or fails |
| **Recovery Impact** | **Blocks health check during software updates** |

---

## 🔍 Recovery Decision Tree

```typescript
// ReaderHealthManager.tsx line 121-179
const determineRecoveryType = ({
  sdkOnline,
  offlineModeEnabled,
  isConnected,
  isReaderOnline,
  isReady,
  terminalLayout,
  readerPaymentStatus,
  transaction
}) => {
  // 1. SDK offline without offline mode enabled
  if (!sdkOnline && !offlineModeEnabled) {
    return { needsRecovery: true, recoveryType: "sdk-offline" };
  }

  // 2. SDK offline with offline mode enabled - this is OK
  if (!sdkOnline && offlineModeEnabled) {
    return { needsRecovery: false, recoveryType: null };
  }

  // 3. Reader disconnected
  if (!isConnected) {
    return { needsRecovery: true, recoveryType: "reader-disconnected" };
  }

  // 4. Reader offline without offline mode (SDK also offline)
  if (!isReaderOnline && !offlineModeEnabled && !sdkOnline) {
    return { needsRecovery: true, recoveryType: "reader-offline" };
  }

  // 5. Reader offline with offline mode enabled - this is OK
  if (!isReaderOnline && offlineModeEnabled) {
    return { needsRecovery: false, recoveryType: null };
  }

  // 6. Tap-to-pay specific check: ONLY "waitingForInput" is healthy
  if (terminalLayout === 'tapToPayLayout') {
    if (readerPaymentStatus !== "waitingForInput") {
      return { needsRecovery: true, recoveryType: "tap-to-pay-not-waiting" };
    }
    return { needsRecovery: false, recoveryType: null };
  }

  // 7. For other layouts, use standard isReady check
  if (!isReady) {
    return { needsRecovery: true, recoveryType: "reader-not-ready" };
  }

  return { needsRecovery: false, recoveryType: null };
};
```

---

## 📋 Quick Reference Table

| Check | Property Path | Healthy | Unhealthy | Recovery Type | Special Handling |
|-------|--------------|---------|-----------|---------------|------------------|
| **Reader Connected** | `readerConnectionStatus` | `"connected"` | Not `"connected"` | `reader-disconnected` | - |
| **Reader Ready** | `terminalData.readerPaymentStatus` | `"ready"` or `"waitingForInput"` | Other values | `reader-not-ready` | - |
| **Reader Online** | `connectedReader.status` | `"online"` | `"offline"` | `reader-offline` | - |
| **SDK Online** | `changeOfflineStatus.sdk.networkStatus` | `"online"` | `"offline"` | `sdk-offline` | - |
| **Offline Mode** | Derived from preferences + SDK | Enabled when needed | Disabled when needed | N/A | - |
| **Payment Session** | `isInPaymentSession` | `true` | `false` | **Blocks health check** | Set by `usePaymentSession` hook |
| **Security Reboot** | `lastDisconnectReason` | `null` or `"disconnectRequested"` | `"securityReboot"` | **Blocks health check for 2 minutes** | ⏳ Wait period |
| **Software Update** | `readerSoftwareUpdateProgress` | `""` (empty) | Non-empty string | **Blocks health check** | - |

---

## 🎯 Health Check Flow

```
performHealthCheck() called every 30 seconds
    ↓
Fetch current state (getConnectedReader, fetchTerminalData)
    ↓
Log health status (comprehensive reader/SDK/payment state)
    ↓
Early Return Checks (in order):
    1. isInPaymentSession !== true? → SKIP (not in payment session)
    2. isHandlingSecurityReboot === true? → SKIP (2-minute wait)
    3. readerSoftwareUpdateProgress !== ""? → SKIP (software update in progress)
    ↓
Check for payment intent timeout/stuck:
    - Payment intent > 60 minutes? → Cancel and reset (TIMEOUT)
    - Payment intent > 50 minutes? → Cancel and reset (PROACTIVE)
    - Payment stuck > 5 minutes? → Cancel and reset (STUCK)
    ↓
Determine if recovery needed (determineRecoveryType):
    - SDK offline (no offline mode)? → "sdk-offline"
    - Reader disconnected? → "reader-disconnected"
    - Reader offline (no offline mode)? → "reader-offline"
    - Tap-to-pay not waiting? → "tap-to-pay-not-waiting"
    - Reader not ready? → "reader-not-ready"
    ↓
Execute recovery with exponential backoff:
    - Fast: 30s → 1m → 2m → 5m (ceiling)
    - Slow: 1m → 2m → 5m → 10m (ceiling)
    - Never gives up (infinite retries)
```

---

## 🔄 Reader Reconnection Mechanisms

The system has **3 independent reconnection paths** that work together to ensure reader connectivity:

### **PATH 1: performHealthCheck() Polling (PRIMARY)**

**Purpose:** Main recovery mechanism that runs continuously

**How it works:**
1. Runs every 30 seconds via `setInterval`
2. Detects reader disconnection via health checks
3. Determines recovery type needed
4. Calls `attemptReaderReconnection()` → `discoverReaders`
5. Uses exponential backoff (30s → 1m → 2m → 5m)

**Code Location:**
```typescript
// ReaderHealthManager.tsx lines 548-554
useEffect(() => {
  const intervalId = setInterval(async () => {
    await performHealthCheck();
  }, pollingIntervalInSeconds * 1000);

  return () => clearInterval(intervalId);
}, [pollingIntervalInSeconds]);
```

**Recovery Flow:**
```
performHealthCheck()
  → determineRecoveryType()
  → attemptRecovery()
  → attemptReaderReconnection()
  → discoverReaders
```

**Critical Dependencies:**
- ✅ Component must be mounted
- ✅ `pollingIntervalInSeconds` must be stable
- ✅ No JavaScript errors in `performHealthCheck()`
- ✅ Early return conditions must pass (kiosk mode, categories, etc.)

**Logs to expect:**
- `"ReaderHealthManager:HealthCheck:"` - Every 30 seconds
- `"Attempting reader reconnection"` - When recovery triggered
- `"RECOVERY SUCCESSFUL"` - When reader reconnects

---

### **PATH 2: Auto-Connect useEffect (SECONDARY)**

**Purpose:** Automatically connect when reader is discovered

**How it works:**
1. Triggered when `discoveredReaders` array changes
2. Checks if reader is disconnected AND locked reader is in discovered list
3. Automatically calls `connectReader`

**Code Location:**
```typescript
// ReaderHealthManager.tsx lines 560-593
useEffect(() => {
  if (
    (isGuidedAccessEnabled === "true" || isGuidedAccessEnabled === true) &&
    (terminalData?.readerPaymentStatus === "notReady" ||
     readerConnectionStatus !== "connected" ||
     changeOfflineStatus?.sdk?.networkStatus !== "online") &&
    !connectingToReader
  ) {
    discoveredReaders.map((reader) => {
      if (reader.serialNumber === lockedReader) {
        postMessageToTerminal("connectReader", {
          serialNumber: lockedReader,
          offlineModePreference: offlineModePreference
        });
      }
    });
  }
}, [discoveredReaders]);
```

**Critical Dependencies:**
- ✅ `discoveredReaders` must be populated (requires PATH 1 to run discovery)
- ✅ `isInPaymentSession` must be true (terminal in payment session)
- ✅ Reader must be disconnected or not ready
- ✅ `lockedReader` must match discovered reader serial number

**Logs to expect:**
- `"ReaderHealthManager:DiscoveredReaderChange:"` - When discoveredReaders changes
- `"ReaderHealthManager:ConnectingToReader:"` - When auto-connect triggers

---

### **PATH 3: Security Reboot Handler (SPECIAL CASE)**

**Purpose:** Handle Stripe M2 security reboots (~every 13 hours)

**How it works:**
1. Detects `lastDisconnectReason === "securityReboot"`
2. Sets `isHandlingSecurityReboot = true`
3. Waits 2 minutes (120 seconds) for reader to finish rebooting
4. Clears `isHandlingSecurityReboot` flag
5. **DOES NOT trigger reconnection directly**
6. **Relies on PATH 1** to detect disconnection and trigger recovery

**Code Location:**
```typescript
// ReaderHealthManager.tsx lines 247-274
useEffect(() => {
  if (lastDisconnectReason !== "securityReboot" || isHandlingSecurityReboot) return;

  LoggingService.warn("Security reboot detected, waiting 2 minutes before reconnection attempt");
  setIsHandlingSecurityReboot(true);

  const timerId = setTimeout(() => {
    LoggingService.info("Security reboot wait complete, ready for reconnection");
    setIsHandlingSecurityReboot(false);
    setRecoveryState(createInitialRecoveryState());
    // Does NOT call attemptReaderReconnection() here!
    // Waits for performHealthCheck() to detect and recover
  }, SECURITY_REBOOT_WAIT); // 2 minutes (120 seconds)

  // ... cleanup
}, [lastDisconnectReason, isHandlingSecurityReboot]);
```

**Critical Dependencies:**
- ✅ `lastDisconnectReason` must be set to "securityReboot"
- ✅ PATH 1 must be running to trigger recovery after 2-minute wait

**Logs to expect:**
- `"Security reboot detected, waiting 2 minutes before reconnection attempt"` - At T+0s
- `"Waiting for security reboot to complete (Xs remaining)"` - Every 30s during wait
- `"Security reboot wait complete, ready for reconnection"` - At T+120s
- Then PATH 1 logs for recovery attempts

---

## ⚠️ Critical Failure Modes

### **Scenario: performHealthCheck() Not Running**

**Symptoms:**
- ❌ No `"ReaderHealthManager:HealthCheck:"` logs
- ❌ Reader stays disconnected indefinitely
- ❌ No recovery attempts
- ❌ `discoveredReaders` stays empty

**Impact:**
- **PATH 1:** Completely broken - no health checks, no recovery
- **PATH 2:** Blocked - no discovered readers to connect to
- **PATH 3:** Blocked - security reboot waits 2 minutes, then nothing happens

**Root Causes:**
1. Component not mounted (ReaderHealthManager not in component tree)
2. Early return condition blocking all checks (but should log warnings)
3. `pollingIntervalInSeconds` changed, causing interval to restart
4. JavaScript error in `performHealthCheck()` before first log statement
5. Interval cleared unexpectedly

**Example from logs/fix30.log:**
```
15:56:01 - Security reboot detected ✅
15:56:01 - "Security reboot detected, waiting 2 minutes..." ✅
15:56:01 to 17:27:10 - SILENCE (91 minutes) ❌
         - Zero "ReaderHealthManager:HealthCheck:" logs
         - Zero recovery attempts
         - Reader stayed disconnected
```

**How to diagnose:**
1. Search logs for `"ReaderHealthManager:HealthCheck:"` - should appear every 30s
2. If zero results, performHealthCheck() is not running
3. Check for early return warnings:
   - `"Skipping health check - Terminal not in paymentLayouts state"`
4. If no warnings, component may not be mounted or interval not started

---

### **Scenario: discoveredReaders Empty**

**Symptoms:**
- ✅ `"ReaderHealthManager:DiscoveredReaderChange:"` logs present
- ❌ `discoveredReaders: []` (empty array)
- ❌ No auto-connect attempts

**Impact:**
- **PATH 1:** May be running but not calling `discoverReaders`
- **PATH 2:** Blocked - no readers to connect to

**Root Causes:**
1. PATH 1 not running (see above)
2. Discovery not triggered by recovery logic
3. Reader not broadcasting (hardware issue)
4. Bluetooth/network issues preventing discovery

---

### **Scenario: Security Reboot Never Completes**

**Symptoms:**
- ✅ `"Security reboot detected, waiting 2 minutes..."` logged
- ❌ No `"Security reboot wait complete"` log after 2 minutes
- ❌ No `"Waiting for security reboot to complete (Xs remaining)"` logs

**Impact:**
- `isHandlingSecurityReboot` may stay `true` forever
- Health checks blocked indefinitely

**Root Causes:**
1. `setTimeout` callback never fired (JavaScript engine issue)
2. Component unmounted before timer completed
3. `lastDisconnectReason` changed before timer completed

---

## 🔍 Debugging Checklist

When reader fails to reconnect, check logs for:

1. **Is performHealthCheck() running?**
   - [ ] Search for `"ReaderHealthManager:HealthCheck:"` - should appear every 30s
   - [ ] If missing, PATH 1 is broken

2. **Are there early return warnings?**
   - [ ] `"Skipping health check - Terminal not in paymentLayouts state"`
   - [ ] `"Waiting for security reboot to complete"`
   - [ ] `"Reader software update in progress"`

3. **Is discovery happening?**
   - [ ] Search for `"Attempting reader reconnection"`
   - [ ] Search for `"discoverReaders"` calls

4. **Are readers being discovered?**
   - [ ] Check `"ReaderHealthManager:DiscoveredReaderChange:"` logs
   - [ ] Verify `discoveredReaders` is not empty

5. **Is auto-connect triggering?**
   - [ ] Search for `"ReaderHealthManager:ConnectingToReader:"`
   - [ ] Verify locked reader matches discovered reader

6. **For security reboots specifically:**
   - [ ] `"Security reboot detected"` at T+0s
   - [ ] `"Waiting for security reboot to complete"` every 30s
   - [ ] `"Security reboot wait complete"` at T+120s
   - [ ] Recovery attempts starting at T+150s (120s wait + 30s first backoff)

---

---

## 🎯 Card Read Timeout & Auto-Recreation (Tap-to-Pay)

### Overview
When a payment intent times out after 60 minutes of waiting for card input, the Stripe SDK automatically cancels the collection and changes the payment status to `"ready"`. For tap-to-pay layouts, the system must automatically create a fresh payment intent to keep the terminal ready for the next transaction.

### The Problem (Fixed in Dec 2024)
**Bug:** When `CardReadTimedOut` error occurred, the payment intent was sometimes already cleared by the Stripe SDK before the error handler ran. This caused the auto-recreation logic to fail silently because:
1. `currentPaymentIntentId` was `null` when error handler checked it
2. The `if (currentPaymentIntentId)` check failed
3. `cancelAndResetPaymentIntent()` was never called
4. Transaction state was never reset to trigger `ReadyToPayScreen` useEffect
5. Terminal remained stuck with no active payment intent

### The Solution
The error handler now **always resets transaction data** for tap-to-pay layouts, regardless of whether the payment intent ID is present. This ensures:
- `paymentProcessingStatus` changes from `"ready"` → `"initialized"`
- `paymentIntentId` changes from `null` → `""`
- `ReadyToPayScreen` useEffect detects the state change and creates a new payment intent

### State Flow

| Time | Event | `paymentIntentId` | `paymentProcessingStatus` | Action |
|------|-------|-------------------|---------------------------|--------|
| T+0 | Payment intent created | `"pi_xxx"` | `"initialized"` → `"readerInput"` | Normal flow |
| T+60min | SDK timeout occurs | `null` (cleared by SDK) | `"ready"` | SDK clears payment intent |
| T+60min+1ms | Error handler runs | `null` | `"ready"` | Detects timeout |
| T+60min+500ms | Transaction reset | `""` | `"initialized"` | State change triggers useEffect |
| T+60min+600ms | New payment intent created | `"pi_yyy"` | `"readerInput"` | Auto-recreation complete |

### Code Implementation

**Error Handler** (`ReactNativeErrorHandler.tsx` lines 86-123):
```typescript
} else if(error.code === "CardReadTimedOut" && error.action === "collectPaymentMethod"){
  LoggingService.warn("⏱️ [Card Read Timeout] Card read timed out after 60 seconds", error?.message);

  if (terminalLayout === 'tapToPayLayout') {
    LoggingService.info("⏱️ [Card Read Timeout] Tap-to-pay mode - resetting to trigger auto-recreation", {
      currentPaymentIntentId: currentPaymentIntentId || "null",
      willCancelPaymentIntent: !!currentPaymentIntentId
    });

    // Cancel if payment intent exists (it may have already been cleared by SDK)
    if (currentPaymentIntentId) {
      postMessageToTerminal("cancelCollectPaymentMethod", currentPaymentIntentId);
    }

    // Always reset transaction data after a delay
    // This changes paymentProcessingStatus from "ready" to "initialized"
    // and triggers ReadyToPayScreen useEffect to auto-create a new payment intent
    setTimeout(() => {
      const resetTransactionData = useGoodbricksTerminalStore.getState().resetTransactionData;
      resetTransactionData();
      LoggingService.info("✅ [Card Read Timeout] Transaction reset complete - ReadyToPayScreen will auto-create payment intent");
    }, 500);
  } else {
    // For other layouts: User will manually create new payment intent
    LoggingService.info("⏱️ [Card Read Timeout] Manual mode - user will create new payment intent");
    if (currentPaymentIntentId) {
      cancelAndResetPaymentIntent(currentPaymentIntentId, "card_read_timeout");
    } else {
      const resetTransactionData = useGoodbricksTerminalStore.getState().resetTransactionData;
      resetTransactionData();
    }
  }
}
```

**Auto-Creation Trigger** (`ReadyToPayScreen.tsx` lines 94-121):
```typescript
useEffect(() => {
  if (paymentProcessingStatus === "processing") {
    send({ type: "cardTapped" });
  } else if (paymentProcessingStatus === "initialized" && (!paymentIntentId || paymentIntentId === "")) {
    // This condition triggers after timeout when state changes to:
    // paymentProcessingStatus = "initialized" (from "ready")
    // paymentIntentId = "" (from null)
    const offlineModeEnabled = offlineModePreference !== "disabled";
    const canCreatePaymentIntent = connectedReader && connectedReader !== "unknown" && (sdkOnline || offlineModeEnabled);

    if (canCreatePaymentIntent) {
      LoggingService.info("ReadyToPayScreen: Creating payment intent", {
        reason: paymentIntentId === "" ? "refresh" : "initial_load",
        sdkOnline,
        offlineModeEnabled
      });
      createPaymentIntent();
    }
  }
}, [paymentProcessingStatus, paymentIntentId, connectedReader, sdkOnline, offlineModePreference]);
```

### Expected Log Sequence

When card read timeout occurs in tap-to-pay mode:

```
[INFO]:[ASYNC]: onDidChangePaymentStatus:  :: "ready"
[ERROR]:[epoch]: collectPaymentMethod error: [object Object] :: {"code":"CardReadTimedOut","message":"Reading the card timed out."}
[INFO]:[FE-epoch]: ReadyToPayScreen: Payment status changed :: "ready"
[WARN]:[FE-epoch]: ⏱️ [Card Read Timeout] Card read timed out after 60 seconds :: "Reading the card timed out."
[INFO]:[FE-epoch]: ⏱️ [Card Read Timeout] Tap-to-pay mode - resetting to trigger auto-recreation :: {"currentPaymentIntentId":"null","willCancelPaymentIntent":false}
[INFO]:[FE-epoch]: ✅ [Card Read Timeout] Transaction reset complete - ReadyToPayScreen will auto-create payment intent
[INFO]:[FE-epoch]: ReadyToPayScreen: Payment status changed :: "initialized"
[INFO]:[FE-epoch]: ReadyToPayScreen: Creating payment intent :: {"reason":"refresh","sdkOnline":true,"offlineModeEnabled":true}
[INFO]:[FE-epoch]: ReadyToPayScreen: Creating payment intent :: {"totalAmount":400,"category":"Donation"}
[INFO]:[ASYNC]: createPaymentIntent :: {...}
[INFO]:[FE-epoch]: ✅ [Payment Intent] Created :: {"paymentIntentId":"pi_xxx","previousPaymentIntentId":""}
```

### Debugging Card Read Timeout Issues

If auto-recreation fails after timeout:

1. **Check error handler logs:**
   - [ ] `"⏱️ [Card Read Timeout] Card read timed out after 60 seconds"`
   - [ ] `"⏱️ [Card Read Timeout] Tap-to-pay mode - resetting to trigger auto-recreation"`
   - [ ] `"✅ [Card Read Timeout] Transaction reset complete"`

2. **Check state changes:**
   - [ ] `"ReadyToPayScreen: Payment status changed :: 'initialized'"` (should appear after reset)
   - [ ] `"ReadyToPayScreen: Creating payment intent :: {\"reason\":\"refresh\"}"` (should appear after state change)

3. **Verify prerequisites:**
   - [ ] `terminalLayout === 'tapToPayLayout'`
   - [ ] `connectedReader !== null && connectedReader !== "unknown"`
   - [ ] `sdkOnline === true` OR `offlineModePreference !== "disabled"`

4. **Common issues:**
   - Missing `"Transaction reset complete"` log → Error handler didn't run or setTimeout didn't fire
   - Missing `"Payment status changed :: 'initialized'"` log → State didn't change or screen not mounted
   - Missing `"Creating payment intent"` log → Prerequisites not met (check reader connection, SDK status, offline mode)

---

## 📍 Key Code Locations

- **Health Check Logic:** `gb2-terminal-web/src/components/ReaderHealthManager.tsx`
- **Error Handler:** `gb2-terminal-web/src/utils/ReactNativeErrorHandler.tsx`
- **Tap-to-Pay Auto-Creation:** `gb2-terminal-web/src/stateMachine/screens/tapToPayLayout/ReadyToPayScreen.tsx`
- **Store Definition:** `gb2-terminal-web/src/utils/GoodbricksTerminalStore.tsx`
- **Native Callbacks:** `gb2-terminal-expo/hooks/useStripeCallbacks.js`
- **Message Bridge:** `gb2-terminal-web/src/components/ReactNativeBridge.tsx`
- **Recovery Documentation:** `DISCONNECTION_TYPES_AND_RECOVERY.md`

