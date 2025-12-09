# Recovery System Quick Reference Card

## 🔍 Health Monitoring

**Polling Interval:** Every 30 seconds  
**Component:** `ReaderHealthManager.tsx`  
**Runs:** Continuously in background (all layouts)

---

## 📊 Monitored Metrics

| Metric | Healthy | Unhealthy | Recovery Type |
|--------|---------|-----------|---------------|
| **Reader Connection** | Connected | Not connected | `reader-disconnected` |
| **Reader Status** | ready/waitingForInput | Other states | `reader-not-ready` |
| **Reader Online** | online | offline | `reader-offline` |
| **SDK Network** | online | offline | `sdk-offline` |
| **Payment Age** | < 50 min | > 50 min | Proactive refresh |
| **Payment Timeout** | < 60 min | > 60 min | Force refresh |
| **Payment Stuck** | < 5 min waiting | > 5 min waiting | Cancel & reset |

---

## ⚡ Recovery Actions by Layout

### Tap-to-Pay Layout
```
Issue Detected → Auto-Reconnect → Auto-Refresh PI → User Stays on Tap Screen
```
**User Experience:** Seamless, invisible recovery

### Single-Page Layout
```
Issue Detected → Auto-Reconnect → User Clicks "Charge" Again
```
**User Experience:** Visible recovery, manual retry

### Multi-Page Layout
```
Issue Detected → Auto-Reconnect → User Clicks "Charge" Again
```
**User Experience:** Visible recovery, manual retry

---

## ⏱️ Backoff Schedules

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

## 🎯 Recovery Type Details

### `reader-not-ready`
- **Trigger:** `readerPaymentStatus ≠ "ready" AND ≠ "waitingForInput"`
- **Backoff:** Fast (30s → 5m)
- **Action:** Full reconnection (disconnect → discover → connect)
- **Auto-refresh PI:** Yes (tap-to-pay only)

### `reader-disconnected`
- **Trigger:** `readerConnectionStatus ≠ "connected"`
- **Backoff:** Fast (30s → 5m)
- **Action:** Full reconnection
- **Auto-refresh PI:** Yes (tap-to-pay only)

### `reader-offline`
- **Trigger:** `connectedReader.status === "offline"` AND SDK offline AND offline mode disabled
- **Backoff:** Fast (30s → 5m)
- **Action:** Full reconnection
- **Auto-refresh PI:** Yes (tap-to-pay only)

### `sdk-offline`
- **Trigger:** `changeOfflineStatus.sdk.networkStatus === "offline"` AND offline mode disabled
- **Backoff:** Slow (1m → 10m)
- **Action:** Wait for internet (no action)
- **Auto-refresh PI:** Yes (tap-to-pay only)

---

## 🔄 Special Recovery Scenarios

### Network Blip (Tap-to-Pay Only)
- **Trigger:** Network status changes while waiting for card tap
- **Detection:** Real-time (not 30s polling)
- **Action:** Cancel PI → Wait 500ms → Recreate PI
- **User Experience:** Seamless, stays on tap screen

### Payment Intent Timeout
- **Trigger:** PI age > 60 minutes
- **Action:** Cancel → Reset → Auto-create (tap-to-pay only)
- **User Experience:** Seamless (tap-to-pay) or manual retry (other layouts)

### Payment Intent Proactive Refresh
- **Trigger:** PI age > 50 minutes
- **Action:** Cancel → Reset → Auto-create (tap-to-pay only)
- **User Experience:** Seamless (tap-to-pay) or manual retry (other layouts)

### Stuck Payment
- **Trigger:** PI in "waitingForInput" for > 5 minutes
- **Action:** Cancel → Reset → Auto-create (tap-to-pay only)
- **User Experience:** Returns to ready state

### Security Reboot
- **Trigger:** Reader disconnect reason = "securityReboot"
- **Action:** Wait 60 seconds → Then attempt reconnection
- **User Experience:** Brief wait, then auto-recovery

---

## 📍 Key Code Locations

### Health Monitoring
- **File:** `gb2-terminal-web/src/components/ReaderHealthManager.tsx`
- **Main function:** `performHealthCheck()` (line 146)
- **Polling setup:** `useEffect()` (line 483)
- **Recovery logic:** `attemptRecovery()` (line 383)

### Network Blip Handling (Tap-to-Pay)
- **File:** `gb2-terminal-web/src/components/PaymentReaderInputAnimation.tsx`
- **Network monitoring:** `useEffect()` (line 46)
- **Recreation logic:** Lines 80-114

### Auto-Refresh After Recovery (Tap-to-Pay)
- **File:** `gb2-terminal-web/src/components/ReaderHealthManager.tsx`
- **Auto-refresh logic:** Lines 329-357

### Payment Intent Creation (Tap-to-Pay)
- **File:** `gb2-terminal-web/src/stateMachine/screens/tapToPayLayout/ReadyToPayScreen.tsx`
- **Auto-creation logic:** Lines 101-121

---

## 🚨 Troubleshooting

### Issue: Recovery keeps retrying indefinitely
**Cause:** Underlying issue not resolved (hardware failure, no internet)  
**Check:** Logs for recovery type, verify hardware/network  
**Solution:** May need manual intervention

### Issue: Payment intent not auto-refreshing after recovery
**Cause:** Not in tap-to-pay layout  
**Expected:** Single/multi-page layouts require manual retry  
**Solution:** User must click "Charge" again

### Issue: Network blip not handled
**Cause:** Not in tap-to-pay layout OR network change during confirmation  
**Expected:** Network blip handling only for tap-to-pay during "waiting for card tap"  
**Solution:** This is expected behavior

### Issue: Too many recovery attempts logged
**Cause:** Exponential backoff ceiling reached (5 or 10 minutes)  
**Expected:** System will keep trying indefinitely  
**Solution:** This is normal for long-term issues

---

## 📝 Log Messages to Watch For

### Health Check
```
LoggingService.info("ReaderHealthManager:HealthCheck:", { ... })
```
**Frequency:** Every 30 seconds  
**Contains:** All health metrics

### Recovery Started
```
LoggingService.warn("Health issue detected: reader-not-ready")
```
**Indicates:** Recovery process initiated

### Recovery Attempt
```
LoggingService.warn("Recovery attempt #1 for reader-not-ready (0 min since first failure)")
```
**Indicates:** Active recovery in progress

### Recovery Successful
```
LoggingService.info("RECOVERY SUCCESSFUL after 2 minutes and 3 attempts!")
```
**Indicates:** System returned to healthy state

### Network Blip Detected
```
LoggingService.warn("🔄 [Network Blip - Tap-to-Pay] Network status changed while waiting for card tap")
```
**Indicates:** Network blip recovery in progress (tap-to-pay only)

### Payment Intent Timeout
```
LoggingService.error("[Payment Intent Timeout] CRITICAL: Timed out after 60 minutes!")
```
**Indicates:** Payment intent exceeded 60-minute limit

### Stuck Payment
```
LoggingService.warn("Payment intent stuck for 5 minutes, canceling")
```
**Indicates:** Payment waiting too long, being canceled

---

## ✅ Testing Checklist

### Test 1: Reader Disconnect
- [ ] Start tap-to-pay mode
- [ ] Disconnect reader (turn off Bluetooth)
- [ ] Verify health check detects disconnect within 30s
- [ ] Verify auto-reconnection starts
- [ ] Verify payment intent auto-refreshed after reconnection
- [ ] Verify user can tap card

### Test 2: Network Drop (Tap-to-Pay)
- [ ] Start tap-to-pay mode
- [ ] Wait for "Tap, Insert or Swipe" screen
- [ ] Turn off WiFi/cellular
- [ ] Verify network blip detected immediately
- [ ] Verify payment intent canceled and recreated
- [ ] Verify user stays on tap screen (no error)
- [ ] Turn on WiFi/cellular
- [ ] Verify payment intent recreated again
- [ ] Verify user can tap card

### Test 3: Payment Timeout
- [ ] Start tap-to-pay mode
- [ ] Wait 50+ minutes (or mock timestamp)
- [ ] Verify proactive refresh triggered
- [ ] Verify new payment intent created
- [ ] Verify user stays on tap screen

### Test 4: Stuck Payment
- [ ] Start tap-to-pay mode
- [ ] Wait 5+ minutes without tapping card
- [ ] Verify stuck payment detected
- [ ] Verify payment intent canceled
- [ ] Verify new payment intent created (tap-to-pay)

### Test 5: Reader Not Ready
- [ ] Simulate reader in weird state
- [ ] Verify health check detects issue within 30s
- [ ] Verify auto-reconnection triggered
- [ ] Verify payment intent auto-refreshed (tap-to-pay)
- [ ] Verify system returns to healthy state

---

## 🎓 Key Concepts

### Polling vs Real-Time Detection
- **Polling (30s):** Reader health, payment intent age, stuck payments
- **Real-Time:** Network blips (tap-to-pay only), payment errors

### Auto-Refresh vs Manual Retry
- **Auto-Refresh:** Tap-to-pay layout only, seamless recovery
- **Manual Retry:** Single/multi-page layouts, user clicks "Charge"

### Exponential Backoff
- **Purpose:** Avoid overwhelming reader/network during recovery
- **Strategy:** Start fast, slow down gradually, reach ceiling
- **Benefit:** Balance between quick recovery and system stability

### Offline Mode Awareness
- **Behavior:** System doesn't trigger recovery when offline mode is expected
- **Check:** `offlineModePreference` and Stripe offline mode status
- **Benefit:** Avoids false alarms during legitimate offline operation

---

## 📚 Related Documentation

- **[RECOVERY_AND_POLLING_MECHANISM.md](./RECOVERY_AND_POLLING_MECHANISM.md)** - Complete documentation with flow diagrams
- **[NETWORK_BLIP_HANDLING.md](./NETWORK_BLIP_HANDLING.md)** - Network blip recovery details

---

**Last Updated:** 2025-12-02  
**Version:** 1.89 (Web) / Build 40 (Native)

