# Logging Reduction Summary

## Overview
Reduced verbose and repetitive logging while preserving all critical debugging information.

## Changes Made

### 1. **ReaderKeepalive Component** (`gb2-terminal-web/src/components/ReaderKeepalive.tsx`)
**Removed:**
- `🔄 [Keepalive] Not running - no payment intent` (fired on every state change)
- `🔄 [Keepalive] Not running - reader not waiting for input` (fired on every state change)
- `🔄 [Keepalive] Not running - no reader connected` (fired on every state change)

**Kept:**
- `🔄 [Keepalive] Starting keepalive pings` (when keepalive starts)
- `🔄 [Keepalive] Sending keepalive ping` (every 30s when active)
- `🔄 [Keepalive] Stopping keepalive pings` (when keepalive stops)

**Impact:** Reduces ~7 log entries per state change to 0 when keepalive is not running.

---

### 2. **ReaderHealthManager Component** (`gb2-terminal-web/src/components/ReaderHealthManager.tsx`)
**Changed:**
- `ReaderHealthManager:DiscoveredReaderChange` - Only logs when there are actual discovered readers (not on empty arrays)
- `ReaderHealthManager:ConnectedReader` - Only logs when reader serial number or status actually changes (not on every connectedReader update from keepalive pings)

**Kept:**
- All health check logs
- All recovery logs
- All error/warning logs

**Impact:** Reduces ~3 log entries every 30 seconds to only when reader state actually changes.

---

### 3. **ReactNativeBridge Component** (`gb2-terminal-web/src/components/ReactNativeBridge.tsx`)
**Removed:**
- `Connected Reader` log for `goodbricks.connectedReader` event (fired every 30s from keepalive)

**Changed:**
- `Reader Software Update Progress` - Only logs when there's actual progress (not empty strings)

**Kept:**
- `Reader Connected` log for `goodbricks.readerConnected` event (actual connection events)
- `Reader Disconnected` log
- All error logs
- All other event logs

**Impact:** Reduces ~1 log entry every 30 seconds + eliminates empty progress logs.

---

### 4. **ReadyToPayScreen Component** (`gb2-terminal-web/src/stateMachine/screens/tapToPayLayout/ReadyToPayScreen.tsx`)
**Changed:**
- `ReadyToPayScreen: Payment status changed` - Only logs meaningful status changes (`processing`, `initialized`, `notReady`)
- No longer logs intermediate states like `removeCard`, `readerInput`, `waitingForInput`, `ready`, `multipleContactlessCardsDetected`

**Kept:**
- All payment intent creation logs
- All error/warning logs
- All card tap logs

**Impact:** Reduces ~10 log entries per payment flow to ~3 meaningful ones.

---

### 5. **Native Layer** (`gb2-terminal-expo/hooks/useStripeReader.js`)
**Removed:**
- `handleGetConnectedReader` log (fired every 30s from keepalive)

**Kept:**
- All reader connection logs
- All reader disconnection logs
- All error logs

**Impact:** Reduces ~1 log entry every 30 seconds.

---

### 6. **Native Layer** (`gb2-terminal-expo/TerminalApp.js`)
**Removed:**
- `readerSoftwareUpdateProgress After finish` debug log (fired every 30s)

**Impact:** Reduces ~1 log entry every 30 seconds.

---

## Total Impact

### Before Changes (per 30-second interval during idle):
- ~15-20 log entries from keepalive and state updates
- ~10 log entries per payment status change
- Multiple empty/redundant logs

### After Changes (per 30-second interval during idle):
- ~3-5 log entries (only meaningful state changes)
- ~3 log entries per payment status change (only meaningful states)
- No empty/redundant logs

### Estimated Reduction:
- **~70-80% reduction in log volume during normal operation**
- **~70% reduction in payment flow logs**
- **100% elimination of empty/useless logs**

---

## What's Preserved

✅ All error logs (LoggingService.error)
✅ All warning logs (LoggingService.warn)
✅ All critical state transitions
✅ All payment intent creation/cancellation logs
✅ All reader connection/disconnection logs
✅ All recovery attempt logs
✅ All health check summary logs
✅ All network blip logs
✅ All timeout handling logs

---

## Debugging Capability

The logging reduction does NOT affect debugging capability because:

1. **State changes are still logged** - just only when they actually change, not on every update
2. **All errors and warnings are preserved** - no reduction in error visibility
3. **Critical events are still logged** - connections, disconnections, payments, recoveries
4. **Keepalive activity is still visible** - start/stop/ping logs are kept
5. **Health checks are still logged** - full health status every 30s is preserved

The changes only remove **redundant, repetitive, and empty logs** that provide no additional debugging value.

