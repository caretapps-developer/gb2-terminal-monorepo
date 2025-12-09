# Network Blip Handling During Payment Intent Creation (Tap-to-Pay Layout Only)

## Overview
This document describes the implementation of seamless network blip detection and recovery during the **tap-to-pay layout** payment flow, specifically between payment intent creation and card tap.

**Important:** This only applies to **tap-to-pay layout** where payment intents are automatically created. Single-page and multi-page layouts create payment intents on-demand when the user clicks "Charge", so there's no window for network blips to occur between creation and card tap.

## Problem Statement
In **tap-to-pay layout**, when a network connection drops **between payment intent creation and card tap** (during the "waiting for card" phase), we want to:
1. **Immediately cancel the current payment intent**
2. **Automatically recreate a new payment intent** (with appropriate offline mode)
3. **Keep the user on the "Tap, Insert or Swipe" screen** (seamless recovery)
4. **Keep offline mode enabled** for handling network drops during confirmation

This is different from:
- Network drops **during confirmation** → Offline mode handles this (payment completes offline)
- Network drops **before payment intent creation** → User sees "Cannot create payment intent" warning
- **Single/multi-page layouts** → Payment intents created on-demand, no automatic recreation needed

## Solution Architecture

### Detection Phase: "Waiting for Card Tap"
The critical window is when:
- Payment intent has been created (`paymentIntentId` exists)
- Reader is waiting for card tap (`paymentProcessingStatus === "readerInput"` or `"waitingForInput"`)
- User hasn't tapped their card yet (`collectPaymentMethod` is blocking/waiting)

### Implementation Location
**File:** `gb2-terminal-web/src/components/PaymentReaderInputAnimation.tsx`

This component is displayed during the "waiting for card tap" phase, making it the perfect place to monitor network status.

### Network Monitoring Logic

```typescript
// Monitor network status during "waiting for card tap" phase (TAP-TO-PAY LAYOUT ONLY)
useEffect(() => {
  // Only handle network blips for tap-to-pay layout
  const isTapToPayLayout = tapToPayConfig && tapToPayConfig.category && tapToPayConfig.fixedAmount;
  if (!isTapToPayLayout) {
    return; // Skip network blip handling for single/multi-page layouts
  }

  const currentSdkStatus = changeOfflineStatus?.sdk?.networkStatus;
  const previousSdkStatus = previousSdkStatusRef.current;

  // Update ref for next render
  previousSdkStatusRef.current = currentSdkStatus;

  // Detect network state change (online → offline OR offline → online)
  const networkChanged = previousSdkStatus && previousSdkStatus !== currentSdkStatus;

  if (networkChanged && paymentIntentId && !isRecreatingPaymentIntentRef.current) {
    // Prevent duplicate recreation attempts
    isRecreatingPaymentIntentRef.current = true;

    LoggingService.warn("🔄 [Network Blip - Tap-to-Pay] Network status changed - recreating payment intent");

    // 1. Cancel the current payment intent
    postMessageToTerminal("cancelCollectPaymentMethod", paymentIntentId);

    // 2. Wait for cancellation to complete, then recreate using tapToPayConfig
    setTimeout(() => {
      const category = tapToPayConfig.category;
      const amount = tapToPayConfig.fixedAmount;

      postMessageToTerminal("createPaymentIntent", {
        totalAmount: amount,
        autoCollect: true,
        offlineModePreference: offlineModePreference,
        paymentIntentMetaData: { /* ... */ },
        relatedInfo: { category: { ...category } }
      });

      // Reset flag after recreation
      setTimeout(() => {
        isRecreatingPaymentIntentRef.current = false;
      }, 2000);
    }, 500); // 500ms delay to ensure cancellation completes
  }
}, [changeOfflineStatus?.sdk?.networkStatus, paymentIntentId, tapToPayConfig, /* ... */]);
```

## User Experience Flow

### Normal Flow (No Network Issues)
1. User selects category/amount
2. Payment intent created with `force_offline` (if offline mode enabled)
3. Reader shows "Tap, Insert or Swipe" screen
4. User taps card
5. Payment processes (online or offline depending on network)
6. Success!

### Network Blip Flow (Network drops before card tap)
1. User selects category/amount
2. Payment intent created with `force_offline` (if offline mode enabled)
3. Reader shows "Tap, Insert or Swipe" screen
4. **Network drops** (wifi → none, or cellular → none)
5. **System detects network drop immediately**
6. **Current payment intent is canceled**
7. **New payment intent is created automatically** (with `require_online` since network is offline)
8. **User stays on "Tap, Insert or Swipe" screen** (seamless recovery)
9. User can tap card when ready
10. Payment processes according to current network state

### Network Drop During Confirmation (Offline Mode Handles This)
1. User selects category/amount
2. Payment intent created with `force_offline`
3. Reader shows "Tap, Insert or Swipe" screen
4. User taps card
5. **Network drops during confirmation**
6. **Offline mode kicks in** - Payment completes offline
7. Payment syncs when network returns
8. Success!

## Seamless Recovery

### Automatic Payment Intent Recreation
The system automatically recreates the payment intent when network status changes, without showing any error to the user.

**Key Features:**
1. **Immediate Detection:** Network state changes are detected in real-time
2. **Automatic Cancellation:** Current payment intent is canceled to prevent errors
3. **Automatic Recreation:** New payment intent is created with appropriate offline mode
4. **No User Interruption:** User stays on "Tap, Insert or Swipe" screen throughout
5. **Duplicate Prevention:** Flag prevents multiple recreation attempts during rapid network fluctuations

## Benefits

### 1. **Seamless User Experience**
- User never sees an error screen
- No interruption to payment flow
- System handles network changes transparently

### 2. **Prevents Payment Errors**
- Avoids "offlineBehavior mismatch" errors
- Prevents unintended offline payments
- Ensures payment intent matches current network state

### 3. **Keeps Offline Mode**
- Offline mode still works for network drops during confirmation
- Automatically adjusts payment intent based on network state
- Best of both worlds: resilience + adaptability

### 4. **Automatic Recovery**
- No user action required
- Works for both online → offline and offline → online transitions
- Handles rapid network fluctuations gracefully

## Testing Scenarios

### Test 1: Network Drop Before Card Tap
1. Start payment flow
2. Wait for "Tap, Insert or Swipe" screen
3. Turn off wifi/cellular
4. **Verify user stays on "Tap, Insert or Swipe" screen** (no error shown)
5. **Verify logs show payment intent canceled and recreated**
6. Wait a moment for recreation to complete
7. Turn on wifi/cellular
8. **Verify user still on "Tap, Insert or Swipe" screen**
9. Tap card and verify payment completes successfully

### Test 2: Network Drop During Confirmation
1. Start payment flow
2. Wait for "Tap, Insert or Swipe" screen
3. Tap card
4. Turn off wifi/cellular during processing
5. Verify payment completes offline
6. Turn on wifi/cellular
7. Verify payment syncs successfully

### Test 3: Rapid Network Fluctuations
1. Start payment flow
2. Wait for "Tap, Insert or Swipe" screen
3. Toggle wifi on/off rapidly
4. Verify system handles gracefully
5. Verify no duplicate payment intents

## Configuration

### Offline Mode Preference
The system respects the `offlineModePreference` setting:
- `"auto"` - Use Stripe account configuration
- `"enabled"` - Force offline mode on (creates payment intents with `force_offline`)
- `"disabled"` - Force offline mode off (creates payment intents with `require_online`)

Network blip detection works regardless of offline mode setting, but the behavior differs:
- **Offline mode enabled:** Cancels payment intent created with `force_offline`
- **Offline mode disabled:** Cancels payment intent created with `require_online`

## Logging

All network blip events are logged with the `🔴 [Network Blip]` prefix for easy filtering:

```
🔴 [Network Blip] Network went offline while waiting for card tap - canceling payment
🗑️ [Network Blip] Canceling payment intent due to network drop
🌐 [PaymentFailure] Network recovered while on error screen - auto-canceling stale payment intent
```

## Related Files

### Modified Files
1. `gb2-terminal-web/src/components/PaymentReaderInputAnimation.tsx`
   - Added network status monitoring
   - Added network blip detection and cancellation logic

2. `gb2-terminal-web/src/components/PaymentFailure.tsx`
   - Added `NETWORK_BLIP_DURING_CARD_WAIT` error code handling
   - Auto-recovery on network restoration

### Related Files (No Changes)
1. `gb2-terminal-web/src/components/ReactNativeBridge.tsx`
   - Receives `goodbricks.changeOfflineStatus` events from native layer
   - Updates Zustand store with network status

2. `gb2-terminal-expo/hooks/useStripeCallbacks.js`
   - Native layer callback for `onDidChangeOfflineStatus`
   - Sends network status updates to WebView

3. `gb2-terminal-expo/hooks/useStripePayment.js`
   - Creates payment intents with `force_offline` or `require_online`
   - Handles offline payment confirmation

## Future Enhancements

### Potential Improvements
1. **Debouncing:** Add 500ms-1s debounce to prevent false positives from rapid network fluctuations
2. **Network Quality:** Detect poor network quality (slow/unstable) and warn user proactively
3. **Retry Logic:** Offer automatic retry when network returns instead of requiring user action
4. **Analytics:** Track frequency of network blips to identify problematic locations/networks
5. **User Notification:** Add visual indicator (banner) when network is unstable

### Considerations
- Current implementation is immediate (no debounce) for fastest response
- May trigger on very brief network fluctuations (<100ms)
- Trade-off: Speed vs. false positives
- Can add debouncing if false positives become an issue

## Conclusion

This implementation provides a robust solution for handling network blips during the critical "waiting for card tap" phase, while preserving offline mode functionality for network drops during payment confirmation. The fail-fast approach with clear error messaging improves user experience and reduces confusion.

