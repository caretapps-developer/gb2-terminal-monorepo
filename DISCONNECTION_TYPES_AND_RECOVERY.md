# Reader & Internet Disconnection Types and Recovery Steps

## Overview
This document outlines all the different types of reader and internet disconnection scenarios handled by the Goodbricks Terminal system, along with their detection methods and recovery strategies.

---

## 🔌 Disconnection Types

### 1. Reader Disconnected
**Detection:**
- `readerConnectionStatus !== "connected"`

**Causes:**
- Bluetooth connection lost
- Reader powered off
- Reader out of range
- Physical disconnection

**Recovery Type:** `reader-disconnected`

**Recovery Steps:**
1. Detect disconnection via `readerConnectionStatus`
2. Set `lastDisconnectReason` and `lastDisconnectTime` in store
3. Cancel any active payment intent
4. Wait for backoff delay (30s → 1m → 2m → 5m → 5m)
5. Attempt full reconnection:
   - Cancel any ongoing discovery
   - Clear discovered readers state
   - Start reader discovery with `readerType`
   - Auto-connect when locked reader is found
6. On success: Auto-refresh payment intent (tap-to-pay only)

**Backoff Schedule:** Fast (30s → 1m → 2m → 5m → 5m max)

---

### 2. Reader Not Ready
**Detection:**
- `readerPaymentStatus !== "ready"` AND `readerPaymentStatus !== "waitingForInput"`

**Causes:**
- Reader in error state
- Reader processing previous transaction
- Reader firmware issue
- Reader initialization incomplete

**Recovery Type:** `reader-not-ready`

**Recovery Steps:**
1. Detect via `readerPaymentStatus` check
2. Wait for backoff delay
3. Attempt full reader reconnection (same as Reader Disconnected)
4. On success: Auto-refresh payment intent (tap-to-pay only)

**Backoff Schedule:** Fast (30s → 1m → 2m → 5m → 5m max)

---

### 3. Reader Offline
**Detection:**
- `connectedReader.status === "offline"` 
- AND `changeOfflineStatus.sdk.networkStatus === "offline"`
- AND `offlineModePreference === "disabled"`

**Causes:**
- Reader can't reach Stripe backend
- Internet connection lost on reader
- Stripe API unreachable
- Network firewall blocking Stripe

**Recovery Type:** `reader-offline`

**Recovery Steps:**
1. Detect via reader status and SDK network status
2. If offline mode is ENABLED: No recovery needed (this is normal operation)
3. If offline mode is DISABLED:
   - Wait for backoff delay
   - Attempt full reader reconnection
   - On success: Auto-refresh payment intent (tap-to-pay only)

**Backoff Schedule:** Fast (30s → 1m → 2m → 5m → 5m max)

**Note:** When offline mode is enabled, reader offline status is expected and healthy.

---

### 4. SDK Offline (Internet Outage)
**Detection:**
- `changeOfflineStatus.sdk.networkStatus === "offline"`
- AND `offlineModePreference === "disabled"`

**Causes:**
- Device internet connection lost
- WiFi disconnected
- Cellular data unavailable
- Network infrastructure down

**Recovery Type:** `sdk-offline`

**Recovery Steps:**
1. Detect via SDK network status
2. If offline mode is ENABLED: No recovery needed (payments continue offline)
3. If offline mode is DISABLED:
   - Wait for backoff delay (slower schedule)
   - Monitor for network restoration
   - No active reconnection needed (passive wait)
   - On network restore: Auto-refresh payment intent (tap-to-pay only)

**Backoff Schedule:** Slow (1m → 2m → 5m → 10m → 10m max)

**Note:** Uses slower backoff since internet outages typically take longer to resolve.

---

### 5. Security Reboot
**Detection:**
- `lastDisconnectReason === "securityReboot"`

**Causes:**
- Reader performing mandatory security update
- Reader firmware security patch
- Stripe-initiated security maintenance

**Recovery Type:** Special handling (not a standard recovery type)

**Recovery Steps:**
1. Detect via `lastDisconnectReason`
2. Set `isHandlingSecurityReboot = true`
3. **Wait 2 minutes (120 seconds)** for reader to complete reboot
4. After 2-minute wait:
   - Reset recovery state
   - Allow normal reconnection to proceed
5. Use standard exponential backoff after initial wait

**Special Handling:**
- Health checks are skipped during the 60-second wait
- Timer is cleared if component unmounts
- Prevents premature reconnection attempts

---

### 6. Unexpected Reader Disconnect
**Detection:**
- Stripe Terminal `onDidDisconnect` callback triggered
- Includes 2-hour session timeout

**Causes:**
- 2-hour Stripe Terminal session timeout
- Reader battery died
- Reader firmware crash
- Bluetooth interference

**Recovery Type:** Triggers `reader-disconnected` recovery

**Recovery Steps:**
1. Stripe Terminal fires `onDidDisconnect` event
2. Native layer sends `goodbricks.disconnect` message to web
3. Web layer:
   - Cancels active payment intent
   - Resets transaction state
   - Sets `lastDisconnectReason` and `lastDisconnectTime`
4. ReaderHealthManager detects disconnection
5. Standard `reader-disconnected` recovery proceeds

---

### 7. Network Blip (Tap-to-Pay Only)
**Detection:**
- `changeOfflineStatus.sdk.networkStatus` changes (online ↔ offline)
- During active payment intent creation/collection

**Causes:**
- Brief WiFi interruption
- Momentary cellular data loss
- Network switching (WiFi ↔ cellular)

**Recovery Type:** Immediate payment intent recreation

**Recovery Steps:**
1. Detect network status change via `changeOfflineStatus`
2. If payment intent exists and network changed:
   - Cancel current payment intent immediately
   - Wait 500ms
   - Create new payment intent (offline or online mode based on current status)
3. No backoff delay (immediate recovery)

**Special Handling:**
- Only applies to tap-to-pay layout
- Prevents stuck payment intents during network transitions
- Uses `isRecreatingPaymentIntentRef` to prevent duplicate attempts

---

### 8. Stuck Payment Intent
**Detection:**
- `paymentAge > 5 minutes`
- AND `readerPaymentStatus === "waitingForInput"`

**Causes:**
- Customer walked away without completing payment
- Reader stuck waiting for card
- Payment flow interrupted

**Recovery Type:** Payment intent cancellation

**Recovery Steps:**
1. Health check detects payment age > 5 minutes
2. Cancel payment intent immediately
3. Reset transaction data
4. Reset recovery state
5. Allow new payment intent creation

**No Backoff:** Immediate cancellation

---

### 9. Payment Intent Timeout
**Detection:**
- **Proactive:** `paymentAge >= 50 minutes`
- **Critical:** `paymentAge >= 60 minutes`

**Causes:**
- Payment intent approaching Stripe's 60-minute expiration
- Long-running kiosk session without completion

**Recovery Type:** Payment intent refresh

**Recovery Steps:**
1. **At 50 minutes (Proactive):**
   - Log warning about approaching timeout
   - Cancel current payment intent
   - Wait 1 second
   - Create new payment intent (tap-to-pay only)

2. **At 60 minutes (Critical):**
   - Log error about timeout
   - Force cancel payment intent
   - Wait 1 second
   - Create new payment intent (tap-to-pay only)

**No Backoff:** Immediate refresh

---

## 🔄 Recovery Mechanisms

### Exponential Backoff Schedules

**Fast Schedule** (for reader issues):
```
Attempt 1: 30 seconds
Attempt 2: 1 minute
Attempt 3: 2 minutes
Attempt 4+: 5 minutes (ceiling)
```

**Slow Schedule** (for internet issues):
```
Attempt 1: 1 minute
Attempt 2: 2 minutes
Attempt 3: 5 minutes
Attempt 4+: 10 minutes (ceiling)
```

### Recovery Actions

**Full Reader Reconnection:**
1. Cancel ongoing discovery
2. Wait 2 seconds
3. Clear discovered readers state
4. Wait 1 second
5. Start discovery with `readerType`
6. Auto-connect when locked reader found

**Payment Intent Refresh (Tap-to-Pay Only):**
1. Cancel current payment intent
2. Wait 500ms
3. Reset transaction data
4. New payment intent auto-created on next render

---

## 📊 Recovery State Tracking

The system tracks recovery state in Zustand store:

```typescript
recoveryState: {
  isRecovering: boolean,        // Currently in recovery mode
  recoveryType: string | null,  // Type of recovery being performed
  attemptCount: number,          // Number of recovery attempts
  lastAttemptTime: number | null, // Timestamp of last attempt
  firstFailureTime: number | null // When issue first detected
}
```

---

## 🎯 Skip Conditions

Recovery is **skipped** when:
- Not in paymentLayouts state (`!currentState.matches("paymentLayouts")`)
- Handling security reboot (2-minute wait period)
- Reader software update in progress

---

## 🔍 Error Codes

### Stripe Terminal Error Codes

| Code | Meaning | Handling |
|------|---------|----------|
| `NotConnectedToReader` | No reader connected | Cancel payment intent, trigger reconnection |
| `AlreadyConnectedToReader` | Already connected | Get current reader, update state |
| `AlreadyDiscovering` | Discovery already in progress | Log warning, continue |
| `CancelFailedAlreadyCompleted` | Cancel called after completion | Log warning, ignore |
| `Canceled` | Operation canceled by user | Log warning, show error |

---

## 📝 Logging

All recovery actions are logged with:
- Recovery type
- Attempt count
- Time since first failure
- Current backoff interval
- Milestone markers (every 10 attempts)

Example log:
```
Recovery attempt #15 for reader-disconnected (45 min since first failure)
Recovery Milestone: 20 attempts over 60 minutes
RECOVERY SUCCESSFUL after 65 minutes and 22 attempts!
```

---

## 🎨 UI Indicators

**TerminalHealthStatus Component** shows:
- 🟢 Green: Healthy, connected
- 🔵 Blue: Operating in offline mode (healthy)
- 🟡 Yellow: Recovering (with attempt count and time)
- 🔴 Red: Critical issue

Recovery status displayed to user:
```
"Reconnecting... (Attempt 3, 2 min elapsed)"
"Operating Offline (5 pending payments)"
```

