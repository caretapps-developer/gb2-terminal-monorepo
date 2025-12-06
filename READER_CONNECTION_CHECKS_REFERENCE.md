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

### 6️⃣ **`isGuidedAccessEnabled`**

| Property | `isGuidedAccessEnabled` |
|----------|------------------------|
| **Type** | `boolean` \| `"true"` \| `"false"` \| `"unknown"` |
| **Source** | Zustand store → Set by native iOS Guided Access check |
| **Possible Values** | `true` \| `false` \| `"true"` \| `"false"` \| `"unknown"` (initial) |
| **Healthy State** | `true` or `"true"` (kiosk mode enabled) |
| **Unhealthy State** | `false`, `"false"`, or `"unknown"` |
| **How It's Evaluated** | `isKioskModeEnabled(isGuidedAccessEnabled) = isGuidedAccessEnabled === "true" \|\| isGuidedAccessEnabled === true` |
| **When Set** | - App load: `initGuidedAccessCheck()` checks iOS Guided Access status<br>- Runtime: iOS event `guidedAccessStatusDidChange` |
| **When Unset** | Guided Access disabled in iOS settings |
| **Recovery Impact** | **Health check ONLY runs if this is true!** |

---

### 7️⃣ **`filteredCategories`**

| Property | `filteredCategories` |
|----------|---------------------|
| **Type** | `Array<Category>` |
| **Source** | Zustand store → Set during terminal configuration |
| **Possible Values** | `[]` (empty) or array of category objects |
| **Healthy State** | `filteredCategories.length > 0` |
| **Unhealthy State** | `filteredCategories.length === 0` |
| **How It's Evaluated** | `if (!filteredCategories?.length) return;` |
| **When Set** | User selects categories during terminal setup |
| **When Unset** | Terminal reset or initial state |
| **Recovery Impact** | **Health check ONLY runs if categories are configured!** |

---

### 8️⃣ **`lastDisconnectReason`** (Security Reboot)

| Property | `lastDisconnectReason` |
|----------|------------------------|
| **Type** | `string` \| `null` |
| **Source** | Zustand store → Set by native Stripe SDK `onDidDisconnect` callback |
| **Possible Values** | `"securityReboot"` \| `"disconnectRequested"` \| `"unknown"` \| `null` (initial) |
| **Healthy State** | `null` or `"disconnectRequested"` (intentional disconnect) |
| **Unhealthy State** | `"securityReboot"` or `"unknown"` (unexpected disconnect) |
| **How It's Evaluated** | `if (lastDisconnectReason === "securityReboot")` → Special 60s wait |
| **When Set** | Native SDK callback `onDidDisconnect(reason)` → Sends `goodbricks.disconnect` with reason |
| **When Unset** | After 60s security reboot wait completes → Set to `null` |
| **Recovery Impact** | **Blocks all health checks for 60 seconds!** |

**Security Reboot Timeline:**
```
T=0s    Reader disconnects
        ├─ onDidDisconnect("securityReboot") fires
        ├─ lastDisconnectReason = "securityReboot"
        ├─ lastDisconnectTime = Date.now()
        ├─ connectedReader = null
        └─ isHandlingSecurityReboot = true

T=0-60s Health checks SKIPPED
        └─ Log: "Waiting for security reboot to complete (Xs remaining)"

T=60s   Wait complete
        ├─ isHandlingSecurityReboot = false
        ├─ lastDisconnectReason = null
        ├─ recoveryState reset
        └─ Normal health checks resume

T=60s+  Standard recovery begins
        ├─ Health check detects reader disconnected
        ├─ Exponential backoff: 30s → 1m → 2m → 5m
        └─ Attempt reconnection (discover → connect)
```

**Why 60 Seconds?**
- Stripe M2 readers take 30-60 seconds to complete security reboot
- Prevents premature reconnection attempts that would fail
- Occurs approximately every 13 hours in production

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
| **Kiosk Mode** | `isGuidedAccessEnabled` | `true` or `"true"` | `false` or `"unknown"` | **Blocks health check** | - |
| **Categories** | `filteredCategories.length` | `> 0` | `=== 0` | **Blocks health check** | - |
| **Security Reboot** | `lastDisconnectReason` | `null` or `"disconnectRequested"` | `"securityReboot"` | **Blocks health check for 60s** | ⏳ Wait period |
| **Software Update** | `readerSoftwareUpdateProgress` | `""` (empty) | Non-empty string | **Blocks health check** | - |

---

## 🎯 Health Check Flow

```
performHealthCheck() called every 30 seconds
    ↓
Fetch current state (getConnectedReader, fetchTerminalData)
    ↓
Log health status
    ↓
Early Return Checks (in order):
    1. isGuidedAccessEnabled !== true? → SKIP
    2. filteredCategories.length === 0? → SKIP
    3. isHandlingSecurityReboot === true? → SKIP (60s wait)
    4. readerSoftwareUpdateProgress !== ""? → SKIP
    ↓
Check for payment intent timeout/stuck
    ↓
Determine if recovery needed
    ↓
Execute recovery with exponential backoff
```

---

## 📍 Key Code Locations

- **Health Check Logic:** `gb2-terminal-web/src/components/ReaderHealthManager.tsx`
- **Store Definition:** `gb2-terminal-web/src/utils/GoodbricksTerminalStore.tsx`
- **Native Callbacks:** `gb2-terminal-expo/hooks/useStripeCallbacks.js`
- **Message Bridge:** `gb2-terminal-web/src/components/ReactNativeBridge.tsx`
- **Recovery Documentation:** `DISCONNECTION_TYPES_AND_RECOVERY.md`

