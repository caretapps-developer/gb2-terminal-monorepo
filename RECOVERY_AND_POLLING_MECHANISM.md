# Recovery and Polling Mechanism Documentation

## Overview

The Goodbricks Terminal implements a comprehensive **health monitoring and automatic recovery system** that continuously polls the terminal state and automatically recovers from various failure scenarios. This document explains how the system works across different terminal layouts.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Terminal Layouts](#terminal-layouts)
3. [Health Monitoring System](#health-monitoring-system)
4. [Recovery Mechanisms](#recovery-mechanisms)
5. [Flow Diagrams](#flow-diagrams)
6. [Layout-Specific Behavior](#layout-specific-behavior)

---

## Architecture Overview

### Components

**1. ReaderHealthManager** (`gb2-terminal-web/src/components/ReaderHealthManager.tsx`)
- Main polling and recovery orchestrator
- Runs every 30 seconds (configurable)
- Monitors reader health, payment intent health, network status
- Triggers automatic recovery actions

**2. ReactNativeBridge** (`gb2-terminal-web/src/components/ReactNativeBridge.tsx`)
- Handles communication between web layer and native layer
- Manages payment intent lifecycle events
- Handles unexpected reader disconnects
- Manages 2-hour reader session timeout

**3. PaymentReaderInputAnimation** (`gb2-terminal-web/src/components/PaymentReaderInputAnimation.tsx`)
- Monitors network status during "waiting for card tap" phase
- Handles network blips (tap-to-pay layout only)
- Automatically recreates payment intents on network changes

**4. ReactNativeErrorHandler** (`gb2-terminal-web/src/utils/ReactNativeErrorHandler.tsx`)
- Centralized error handling
- Detects specific error codes (timeouts, disconnects, network issues)
- Triggers appropriate recovery actions

---

## Terminal Layouts

The system supports three terminal layouts, each with different payment intent creation patterns:

### 1. **Tap-to-Pay Layout**
- **Payment Intent Creation:** Automatic (on screen load)
- **User Interaction:** Minimal - just tap card
- **Recovery Needs:** High - payment intents auto-created and must stay fresh
- **Network Blip Handling:** Yes - automatic recreation

### 2. **Single-Page Layout**
- **Payment Intent Creation:** On-demand (user selects category, enters amount, clicks "Charge")
- **User Interaction:** Medium - select category, enter amount
- **Recovery Needs:** Medium - payment intent created right before card tap
- **Network Blip Handling:** No - too short a window

### 3. **Multi-Page Layout**
- **Payment Intent Creation:** On-demand (user navigates through screens, clicks "Charge")
- **User Interaction:** High - multiple screens
- **Recovery Needs:** Medium - payment intent created right before card tap
- **Network Blip Handling:** No - too short a window

---

## Health Monitoring System

### Polling Interval

**Default:** Every 30 seconds

```typescript
export const ReaderHealthManager = ({ pollingIntervalInSeconds = 30 }) => {
  useEffect(() => {
    const intervalId = setInterval(async () => {
      await performHealthCheck();
    }, pollingIntervalInSeconds * 1000);
    
    return () => clearInterval(intervalId);
  });
}
```

### Health Check Process

Every 30 seconds, the system:

1. **Fetches current state** from native layer
2. **Evaluates health metrics**
3. **Detects issues**
4. **Triggers recovery** if needed
5. **Logs status** for monitoring

### Monitored Metrics

| Metric | Source | Healthy State | Unhealthy State |
|--------|--------|---------------|-----------------|
| **Reader Connection** | `readerConnectionStatus` | `"connected"` | Not connected |
| **Reader Payment Status** | `terminalData.readerPaymentStatus` | `"ready"` or `"waitingForInput"` | Any other state |
| **Reader Online Status** | `connectedReader.status` | `"online"` | `"offline"` |
| **SDK Network Status** | `changeOfflineStatus.sdk.networkStatus` | `"online"` | `"offline"` |
| **Payment Intent Age** | `transaction.paymentIntentCreatedAt` | < 50 minutes | > 50 minutes |
| **Payment Stuck Time** | `transaction.paymentIntentCreatedAt` | < 5 minutes in "waitingForInput" | > 5 minutes |

---

## Recovery Mechanisms

### 1. Reader Not Ready Recovery

**Trigger:** `readerPaymentStatus` is NOT `"ready"` or `"waitingForInput"`

**Action:**
```
1. Detect issue → Set recoveryType = "reader-not-ready"
2. Wait for backoff delay (30s → 1m → 2m → 5m → 5m)
3. Attempt reader reconnection:
   - Disconnect reader
   - Discover readers
   - Reconnect to locked reader
4. On success → Auto-refresh payment intent (tap-to-pay only)
```

### 2. Reader Disconnected Recovery

**Trigger:** `readerConnectionStatus !== "connected"`

**Action:**
```
1. Detect issue → Set recoveryType = "reader-disconnected"
2. Wait for backoff delay
3. Attempt reader reconnection
4. On success → Auto-refresh payment intent (tap-to-pay only)
```

### 3. Reader Offline Recovery

**Trigger:** `connectedReader.status === "offline"` AND SDK offline AND offline mode disabled

**Action:**
```
1. Detect issue → Set recoveryType = "reader-offline"
2. Wait for backoff delay
3. Full reader reconnection (disconnect → discover → reconnect)
4. On success → Auto-refresh payment intent (tap-to-pay only)
```

### 4. SDK Offline Recovery

**Trigger:** `changeOfflineStatus.sdk.networkStatus === "offline"` AND offline mode disabled

**Action:**
```
1. Detect issue → Set recoveryType = "sdk-offline"
2. Wait for backoff delay (slower: 1m → 2m → 5m → 10m → 10m)
3. Wait for internet connection to restore (no action needed)
4. On success → Auto-refresh payment intent (tap-to-pay only)
```

### 5. Stuck Payment Recovery

**Trigger:** Payment intent in "waitingForInput" for > 5 minutes

**Action:**
```
1. Log warning
2. Cancel payment intent
3. Reset transaction data
4. Reset recovery state
5. System returns to ready state
```

### 6. Payment Intent Timeout Recovery

**Trigger:** Payment intent age > 60 minutes

**Action:**
```
1. Log critical error
2. Cancel old payment intent
3. Wait 1 second
4. Reset transaction data
5. New payment intent auto-created (tap-to-pay only)
```

### 7. Payment Intent Proactive Refresh

**Trigger:** Payment intent age > 50 minutes (before 60-minute timeout)

**Action:**
```
1. Log warning
2. Cancel old payment intent
3. Wait 1 second
4. Reset transaction data
5. New payment intent auto-created (tap-to-pay only)
```

### 8. Network Blip Recovery (Tap-to-Pay Only)

**Trigger:** Network status changes while waiting for card tap

**Action:**
```
1. Detect network change (online ↔ offline)
2. Cancel current payment intent
3. Wait 500ms for cancellation
4. Recreate payment intent with current network state
5. User stays on "Tap, Insert or Swipe" screen (seamless)
```

---

## Exponential Backoff

The system uses exponential backoff to avoid overwhelming the reader/network:

### Fast Recovery (Reader Issues)
```
Attempt 1: 30 seconds
Attempt 2: 1 minute
Attempt 3: 2 minutes
Attempt 4+: 5 minutes (ceiling)
```

### Slow Recovery (Network Issues)
```
Attempt 1: 1 minute
Attempt 2: 2 minutes
Attempt 3: 5 minutes
Attempt 4+: 10 minutes (ceiling)
```

---

## Flow Diagrams

### Main Health Check Flow

```
┌─────────────────────────────────────────┐
│   Every 30 Seconds: Health Check        │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Fetch Current State from Native Layer  │
│  - Reader connection status              │
│  - Reader payment status                 │
│  - SDK network status                    │
│  - Payment intent age                    │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Check Prerequisites                     │
│  - Guided Access enabled?                │
│  - Categories configured?                │
│  - Security reboot in progress?          │
│  - Software update in progress?          │
└─────────────────┬───────────────────────┘
                  │
                  ▼
         ┌────────┴────────┐
         │  Prerequisites   │
         │     Met?         │
         └────┬────────┬────┘
              │ No     │ Yes
              │        │
              ▼        ▼
          [Skip]   [Continue]
                       │
                       ▼
┌─────────────────────────────────────────┐
│  Check Payment Intent Health             │
│  - Age > 60 min? → Cancel & reset        │
│  - Age > 50 min? → Proactive refresh     │
│  - Stuck > 5 min? → Cancel & reset       │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Evaluate Health Metrics                 │
│  - Reader connected?                     │
│  - Reader ready/waitingForInput?         │
│  - Reader online?                        │
│  - SDK online?                           │
│  - Offline mode enabled?                 │
└─────────────────┬───────────────────────┘
                  │
                  ▼
         ┌────────┴────────┐
         │   All Healthy?   │
         └────┬────────┬────┘
              │ Yes    │ No
              │        │
              ▼        ▼
    ┌─────────────┐  ┌──────────────────┐
    │  Recovery   │  │  Trigger Recovery │
    │  Successful?│  │  - Set recovery   │
    │             │  │    type           │
    └──────┬──────┘  │  - Apply backoff  │
           │         │  - Execute action │
           │ Yes     └──────────┬─────────┘
           │                    │
           ▼                    │
    ┌──────────────────┐        │
    │ Auto-refresh PI? │        │
    │ (Tap-to-Pay only)│        │
    └──────┬───────────┘        │
           │                    │
           ▼                    │
    ┌──────────────────┐        │
    │ Reset recovery   │        │
    │ state            │        │
    └──────────────────┘        │
                                │
                                ▼
                        [Wait 30s, repeat]
```

### Reader Not Ready Recovery Flow

```
┌─────────────────────────────────────────┐
│  Health Check Detects:                   │
│  readerPaymentStatus ≠ "ready" AND       │
│  readerPaymentStatus ≠ "waitingForInput" │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Set Recovery State                      │
│  - recoveryType: "reader-not-ready"      │
│  - attemptCount: 0                       │
│  - firstFailureTime: now                 │
└─────────────────┬───────────────────────┘
                  │
                  ▼
         ┌────────┴────────┐
         │  Wait for Backoff│
         │  Delay           │
         └────┬─────────────┘
              │
              │ Attempt 1: 30s
              │ Attempt 2: 1m
              │ Attempt 3: 2m
              │ Attempt 4+: 5m
              │
              ▼
┌─────────────────────────────────────────┐
│  Attempt Reader Reconnection             │
│  1. Disconnect reader                    │
│  2. Start reader discovery               │
│  3. Wait for locked reader               │
│  4. Connect to locked reader             │
└─────────────────┬───────────────────────┘
                  │
                  ▼
         ┌────────┴────────┐
         │   Reconnection   │
         │   Successful?    │
         └────┬────────┬────┘
              │ No     │ Yes
              │        │
              ▼        ▼
    ┌─────────────┐  ┌──────────────────────┐
    │ Increment   │  │  Check Terminal Layout│
    │ attemptCount│  └──────────┬───────────┘
    │ Wait for    │             │
    │ next cycle  │             ▼
    └─────────────┘    ┌────────┴────────┐
                       │  Tap-to-Pay?    │
                       └────┬────────┬───┘
                            │ No     │ Yes
                            │        │
                            ▼        ▼
                    ┌───────────┐  ┌─────────────────┐
                    │ Log       │  │ Auto-refresh PI │
                    │ success   │  │ 1. Cancel old PI│
                    │ Done      │  │ 2. Reset data   │
                    └───────────┘  │ 3. New PI auto- │
                                   │    created      │
                                   └─────────────────┘
                                            │
                                            ▼
                                   ┌─────────────────┐
                                   │ User stays on   │
                                   │ "Tap, Insert or │
                                   │ Swipe" screen   │
                                   └─────────────────┘
```

### Payment Intent Timeout Flow

```
┌─────────────────────────────────────────┐
│  Health Check Detects:                   │
│  Payment Intent Age > 50 minutes         │
│  (Proactive Refresh)                     │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Log Warning                             │
│  "Approaching 60-minute timeout"         │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Cancel Old Payment Intent               │
│  postMessageToTerminal(                  │
│    "cancelCollectPaymentMethod",         │
│    paymentIntentId                       │
│  )                                       │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Wait 1 Second                           │
│  (Ensure cancellation completes)         │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Reset Transaction Data                  │
│  - paymentIntentId: ""                   │
│  - paymentProcessingStatus: "initialized"│
└─────────────────┬───────────────────────┘
                  │
                  ▼
         ┌────────┴────────┐
         │  Terminal Layout?│
         └────┬────────┬────┘
              │        │
    Tap-to-Pay│        │Single/Multi-Page
              │        │
              ▼        ▼
    ┌─────────────┐  ┌──────────────────┐
    │ ReadyToPay  │  │ User must click  │
    │ Screen      │  │ "Charge" to      │
    │ detects     │  │ create new PI    │
    │ empty PI    │  └──────────────────┘
    │ and auto-   │
    │ creates new │
    │ payment     │
    │ intent      │
    └──────┬──────┘
           │
           ▼
    ┌──────────────────┐
    │ New PI created   │
    │ with autoCollect │
    │ Reader waits for │
    │ card tap         │
    └──────────────────┘
```

### Network Blip Recovery Flow (Tap-to-Pay Only)

```
┌─────────────────────────────────────────┐
│  User on "Tap, Insert or Swipe" Screen   │
│  Payment Intent: pi_abc123               │
│  Network Status: Online                  │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Network Status Changes                  │
│  Online → Offline (or vice versa)        │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  PaymentReaderInputAnimation Detects     │
│  - previousStatus ≠ currentStatus        │
│  - paymentIntentId exists                │
│  - isTapToPayLayout = true               │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Set Recreation Flag                     │
│  isRecreatingPaymentIntentRef = true     │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Cancel Current Payment Intent           │
│  postMessageToTerminal(                  │
│    "cancelCollectPaymentMethod",         │
│    "pi_abc123"                           │
│  )                                       │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Wait 500ms                              │
│  (Ensure cancellation completes)         │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Recreate Payment Intent                 │
│  Using tapToPayConfig:                   │
│  - category                              │
│  - fixedAmount                           │
│  - offlineModePreference (current)       │
│  - autoCollect: true                     │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Native Layer Creates New PI             │
│  - New PI ID: pi_xyz789                  │
│  - Automatically starts collectPayment   │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  User Experience                         │
│  ✅ Stays on "Tap, Insert or Swipe"     │
│  ✅ No error shown                       │
│  ✅ No interruption                      │
│  ✅ Can tap card immediately             │
└─────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Reset Recreation Flag                   │
│  (After 2 seconds)                       │
│  isRecreatingPaymentIntentRef = false    │
└─────────────────────────────────────────┘
```

---

## Layout-Specific Behavior

### Tap-to-Pay Layout

**Characteristics:**
- Payment intents auto-created on screen load
- Reader continuously waits for card taps
- Minimal user interaction required
- High automation, high recovery needs

**Recovery Behaviors:**

| Scenario | Action | User Experience |
|----------|--------|-----------------|
| Reader not ready | Auto-reconnect → Auto-refresh PI | Seamless, stays on ready screen |
| Reader disconnected | Auto-reconnect → Auto-refresh PI | Seamless, stays on ready screen |
| Network blip (before tap) | Cancel → Recreate PI | Seamless, stays on tap screen |
| Payment stuck > 5 min | Cancel → Reset → Auto-create PI | Returns to ready screen |
| PI timeout > 60 min | Cancel → Reset → Auto-create PI | Seamless, stays on tap screen |
| Recovery successful | Auto-refresh PI | Seamless, stays on tap screen |

**Key Code Locations:**
- Auto-creation: `ReadyToPayScreen.tsx` (lines 101-121)
- Network blip handling: `PaymentReaderInputAnimation.tsx` (lines 46-125)
- Auto-refresh after recovery: `ReaderHealthManager.tsx` (lines 329-357)

**Flow:**
```
[Screen Load]
     ↓
[Auto-create PI with autoCollect=true]
     ↓
[Reader waits for card tap]
     ↓
[Health monitoring every 30s]
     ↓
[Issue detected?] → [Auto-recovery] → [Auto-refresh PI] → [Back to waiting]
     ↓ No issue
[User taps card]
     ↓
[Process payment]
```

### Single-Page Layout

**Characteristics:**
- User selects category and enters amount on one screen
- Payment intent created when user clicks "Charge"
- Short window between PI creation and card tap
- Medium user interaction

**Recovery Behaviors:**

| Scenario | Action | User Experience |
|----------|--------|-----------------|
| Reader not ready | Auto-reconnect | User sees reconnection, can retry |
| Reader disconnected | Auto-reconnect | User sees reconnection, can retry |
| Network blip (before tap) | No special handling | Too short a window |
| Payment stuck > 5 min | Cancel → Reset | Returns to category screen |
| PI timeout > 60 min | Cancel → Reset | Returns to category screen |
| Recovery successful | No auto-refresh | User clicks "Charge" again |

**Key Code Locations:**
- PI creation: `SelectCategoryAndAmountScreen.tsx`
- No network blip handling (not needed)
- No auto-refresh after recovery

**Flow:**
```
[Category Screen]
     ↓
[User selects category & enters amount]
     ↓
[User clicks "Charge"]
     ↓
[Create PI with autoCollect=true]
     ↓
[Reader waits for card tap] ← Very short window
     ↓
[User taps card immediately]
     ↓
[Process payment]
```

### Multi-Page Layout

**Characteristics:**
- User navigates through multiple screens
- Payment intent created when user clicks "Charge" on final screen
- Short window between PI creation and card tap
- High user interaction

**Recovery Behaviors:**

| Scenario | Action | User Experience |
|----------|--------|-----------------|
| Reader not ready | Auto-reconnect | User sees reconnection, can retry |
| Reader disconnected | Auto-reconnect | User sees reconnection, can retry |
| Network blip (before tap) | No special handling | Too short a window |
| Payment stuck > 5 min | Cancel → Reset | Returns to category screen |
| PI timeout > 60 min | Cancel → Reset | Returns to category screen |
| Recovery successful | No auto-refresh | User clicks "Charge" again |

**Key Code Locations:**
- PI creation: `EnterAmountScreen.tsx`
- No network blip handling (not needed)
- No auto-refresh after recovery

**Flow:**
```
[Select Category Screen]
     ↓
[User selects category]
     ↓
[Enter Amount Screen]
     ↓
[User enters amount & clicks "Charge"]
     ↓
[Create PI with autoCollect=true]
     ↓
[Reader waits for card tap] ← Very short window
     ↓
[User taps card immediately]
     ↓
[Process payment]
```

---

## Comparison Table

| Feature | Tap-to-Pay | Single-Page | Multi-Page |
|---------|------------|-------------|------------|
| **PI Creation** | Automatic | On-demand | On-demand |
| **Auto-refresh after recovery** | ✅ Yes | ❌ No | ❌ No |
| **Network blip handling** | ✅ Yes | ❌ No | ❌ No |
| **Reader reconnection** | ✅ Auto | ✅ Auto | ✅ Auto |
| **Stuck payment handling** | ✅ Auto-cancel | ✅ Auto-cancel | ✅ Auto-cancel |
| **PI timeout handling** | ✅ Auto-refresh | ✅ Auto-cancel | ✅ Auto-cancel |
| **User intervention needed** | Minimal | Medium | Medium |
| **Recovery transparency** | Seamless | Visible | Visible |

---

## Key Takeaways

### 1. **Global Polling Runs for All Layouts**
The `ReaderHealthManager` runs every 30 seconds regardless of layout, monitoring:
- Reader connection status
- Reader payment status
- Payment intent health
- Network status

### 2. **Recovery Actions Differ by Layout**
- **Tap-to-Pay:** Fully automatic recovery with payment intent auto-refresh
- **Single/Multi-Page:** Automatic reader reconnection, but user must manually create new payment intent

### 3. **Network Blip Handling is Tap-to-Pay Only**
Because tap-to-pay has a long window between PI creation and card tap, network blips are handled seamlessly. Single/multi-page layouts have too short a window for this to be an issue.

### 4. **Exponential Backoff Prevents Overwhelming**
The system uses exponential backoff (30s → 1m → 2m → 5m) to avoid hammering the reader/network during recovery.

### 5. **Offline Mode is Respected**
The system respects offline mode settings and doesn't trigger recovery when operating in offline mode is expected.

### 6. **User Experience Priority**
- **Tap-to-Pay:** Seamless, invisible recovery
- **Single/Multi-Page:** Visible recovery, but automatic reconnection reduces user burden

---

## Configuration

### Polling Interval
Default: 30 seconds (configurable via prop)

```typescript
<ReaderHealthManager pollingIntervalInSeconds={30} />
```

### Backoff Schedules
```typescript
// Fast recovery (reader issues)
const BACKOFF_SCHEDULE = [30, 60, 120, 300, 300]; // seconds

// Slow recovery (network issues)
const SLOW_BACKOFF_SCHEDULE = [60, 120, 300, 600, 600]; // seconds
```

### Timeout Thresholds
```typescript
const PAYMENT_INTENT_TIMEOUT = 60 * 60 * 1000; // 60 minutes
const PAYMENT_INTENT_REFRESH_THRESHOLD = 50 * 60 * 1000; // 50 minutes
const PAYMENT_STUCK_THRESHOLD = 5 * 60 * 1000; // 5 minutes
const SECURITY_REBOOT_WAIT = 60 * 1000; // 60 seconds
```

---

## Monitoring and Logging

All recovery actions are logged with detailed context:

```typescript
LoggingService.info("ReaderHealthManager:HealthCheck:", {
  terminalName,
  connectedReader,
  isConnected,
  isReady,
  sdkOnline,
  readerPaymentStatus,
  recoveryType,
  attemptCount,
  timeSinceFirstFailure,
  // ... more metrics
});
```

**Log Levels:**
- `INFO` - Normal health checks, successful recoveries
- `WARN` - Issues detected, recovery attempts
- `ERROR` - Critical failures, timeouts

---

## Testing Scenarios

### Test 1: Reader Disconnect During Tap-to-Pay
1. Start tap-to-pay mode
2. Wait for "Tap, Insert or Swipe" screen
3. Disconnect reader (turn off Bluetooth)
4. **Expected:** Health check detects disconnect within 30s
5. **Expected:** Auto-reconnection starts
6. **Expected:** After reconnection, payment intent auto-refreshed
7. **Expected:** User stays on tap screen, can tap card

### Test 2: Network Drop During Tap-to-Pay
1. Start tap-to-pay mode
2. Wait for "Tap, Insert or Swipe" screen
3. Turn off WiFi/cellular
4. **Expected:** Network blip detected immediately
5. **Expected:** Payment intent canceled and recreated
6. **Expected:** User stays on tap screen (seamless)
7. Turn on WiFi/cellular
8. **Expected:** Payment intent recreated again with online mode
9. **Expected:** User can tap card successfully

### Test 3: Payment Intent Timeout
1. Start tap-to-pay mode
2. Wait for "Tap, Insert or Swipe" screen
3. Wait 50+ minutes (or mock the timestamp)
4. **Expected:** Health check detects approaching timeout
5. **Expected:** Payment intent proactively refreshed
6. **Expected:** User stays on tap screen

### Test 4: Stuck Payment
1. Start tap-to-pay mode
2. Create payment intent
3. Wait 5+ minutes without tapping card
4. **Expected:** Health check detects stuck payment
5. **Expected:** Payment intent canceled
6. **Expected:** Transaction reset
7. **Expected:** New payment intent auto-created (tap-to-pay)

### Test 5: Reader Not Ready
1. Start tap-to-pay mode
2. Simulate reader in weird state (not "ready" or "waitingForInput")
3. **Expected:** Health check detects issue within 30s
4. **Expected:** Auto-reconnection triggered
5. **Expected:** After reconnection, payment intent auto-refreshed
6. **Expected:** System returns to healthy state

---

## Troubleshooting

### Issue: Recovery Loop (Keeps Retrying)
**Cause:** Underlying issue not resolved (e.g., reader hardware failure, no internet)
**Solution:** Check logs for recovery type, verify hardware/network, may need manual intervention

### Issue: Payment Intent Not Auto-Refreshing
**Cause:** Not in tap-to-pay layout
**Solution:** This is expected behavior for single/multi-page layouts - user must click "Charge"

### Issue: Network Blip Not Handled
**Cause:** Not in tap-to-pay layout, or network change happened during confirmation
**Solution:** Network blip handling only applies to tap-to-pay during "waiting for card tap" phase

### Issue: Too Many Recovery Attempts
**Cause:** Exponential backoff ceiling reached (5 or 10 minutes)
**Solution:** This is expected for long-term issues - system will keep trying indefinitely

---

## Future Enhancements

1. **Configurable Thresholds:** Allow customization of timeout/stuck thresholds per organization
2. **Recovery Metrics Dashboard:** Track recovery success rates, common failure modes
3. **Predictive Recovery:** Detect patterns and proactively recover before failures
4. **User Notifications:** Optional notifications for long-running recoveries
5. **Manual Recovery Trigger:** Allow users to manually trigger recovery from UI

---

## Visual Diagrams

The following interactive Mermaid diagrams are available to visualize the recovery flows:

### 1. Complete Health Check and Recovery Flow
Shows the complete 30-second polling cycle with all decision points and recovery paths.

### 2. Tap-to-Pay Layout: Network Blip Recovery
Sequence diagram showing seamless network blip handling with payment intent recreation.

### 3. Reader Reconnection Recovery Flow
Detailed sequence diagram of reader disconnection detection and automatic reconnection.

### 4. Layout Comparison: Payment Intent Lifecycle
Side-by-side comparison of how payment intents are created and managed across all three layouts.

### 5. Exponential Backoff Strategy
Timeline visualization showing recovery attempt intervals for fast and slow recovery modes.

---

## Quick Reference

### When Does Auto-Refresh Happen?

| Scenario | Tap-to-Pay | Single-Page | Multi-Page |
|----------|------------|-------------|------------|
| Reader reconnected | ✅ Auto-refresh | ❌ Manual | ❌ Manual |
| Network restored | ✅ Auto-refresh | ❌ Manual | ❌ Manual |
| PI timeout (60 min) | ✅ Auto-refresh | ❌ Manual | ❌ Manual |
| PI proactive (50 min) | ✅ Auto-refresh | ❌ Manual | ❌ Manual |
| Stuck payment (5 min) | ✅ Auto-refresh | ❌ Manual | ❌ Manual |
| Network blip | ✅ Auto-recreate | N/A | N/A |

### Recovery Types

| Type | Trigger | Backoff | Action |
|------|---------|---------|--------|
| `reader-not-ready` | Status ≠ ready/waitingForInput | Fast (30s-5m) | Reconnect |
| `reader-disconnected` | Not connected | Fast (30s-5m) | Reconnect |
| `reader-offline` | Reader status = offline | Fast (30s-5m) | Reconnect |
| `sdk-offline` | SDK network = offline | Slow (1m-10m) | Wait |

### Key Thresholds

| Threshold | Value | Action |
|-----------|-------|--------|
| Health check interval | 30 seconds | Poll reader status |
| Payment stuck | 5 minutes | Cancel & reset |
| Payment proactive refresh | 50 minutes | Cancel & recreate |
| Payment timeout | 60 minutes | Cancel & recreate |
| Security reboot wait | 60 seconds | Wait before reconnect |
| Fast backoff ceiling | 5 minutes | Max wait between attempts |
| Slow backoff ceiling | 10 minutes | Max wait between attempts |

---

## Related Documentation

- [Network Blip Handling](./NETWORK_BLIP_HANDLING.md) - Detailed network blip recovery for tap-to-pay
- [Offline Mode](./OFFLINE_MODE.md) - Offline payment processing (if exists)
- [Reader Session Management](./READER_SESSION_MANAGEMENT.md) - 2-hour session timeout handling (if exists)

---

## Summary

The Goodbricks Terminal implements a **robust, self-healing system** that:

1. ✅ **Continuously monitors** terminal health every 30 seconds
2. ✅ **Automatically detects** reader disconnections, network issues, stuck payments, and timeouts
3. ✅ **Intelligently recovers** using exponential backoff to avoid overwhelming the system
4. ✅ **Seamlessly handles** network blips in tap-to-pay mode without user interruption
5. ✅ **Auto-refreshes** payment intents in tap-to-pay mode after successful recovery
6. ✅ **Respects** offline mode settings and doesn't trigger false alarms
7. ✅ **Logs extensively** for monitoring and troubleshooting
8. ✅ **Runs indefinitely** until the kiosk is healthy again

**For tap-to-pay kiosks**, the system provides a **completely hands-off experience** where the terminal automatically recovers from virtually any failure scenario and returns to a ready state without user intervention.

**For single/multi-page layouts**, the system provides **automatic reader reconnection** while requiring users to manually retry payment creation, which is appropriate given the on-demand nature of these layouts.

---

**Last Updated:** 2025-12-02
**Version:** 1.89 (Web) / Build 40 (Native)
**Author:** Goodbricks Engineering Team


