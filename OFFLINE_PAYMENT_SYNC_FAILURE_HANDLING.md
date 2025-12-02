# Offline Payment Sync Failure Handling

## Current State: ⚠️ NOT IMPLEMENTED

The system **does not currently handle failed offline payment syncs**. This document outlines the problem and recommended solutions.

---

## What Stripe Terminal SDK Provides

According to Stripe's documentation and the sample code, the SDK provides these callbacks:

### 1. `onDidChangeOfflineStatus` ✅ **Implemented**
- **Status:** Currently implemented in `gb2-terminal-expo/hooks/useStripeCallbacks.js` (lines 115-136)
- **Purpose:** Tracks `offlinePaymentsCount` and network status
- **Current Use:** Shows when payments are syncing, updates the UI badge (P0, P1, P3, etc.)

### 2. `didForwardPaymentIntent` ❌ **NOT Implemented**
- **Status:** Not currently listening to this event
- **Purpose:** Called when each payment successfully syncs to Stripe
- **Missing Functionality:** No confirmation when individual payments sync successfully

### 3. `didFailForwardingPaymentIntent` ❌ **NOT Implemented**
- **Status:** Not currently listening to this event
- **Purpose:** Called when a payment **fails** to sync
- **Missing Functionality:** No error handling when sync fails

---

## The Problem

### What Happens When an Offline Payment Fails to Sync?

Currently, if an offline payment fails to sync:

❌ **No error is shown to the user**  
❌ **No notification or alert**  
❌ **The payment stays in the pending queue indefinitely**  
❌ **The `offlinePaymentsCount` might not decrease**  
❌ **The badge will keep showing P1, P2, P3... forever**  
❌ **No way to identify which specific payment failed**  
❌ **No retry mechanism**  

### Possible Reasons for Sync Failure

1. **Payment intent expired** (60-minute limit from creation)
2. **Network issues during sync** (intermittent connectivity)
3. **Stripe API errors** (rate limits, service issues)
4. **Card declined after offline capture** (insufficient funds discovered during sync)
5. **Invalid payment intent state** (already canceled, refunded, etc.)
6. **Reader/SDK errors** (device issues, SDK bugs)

---

## Recommended Solution

### Implementation Steps

Add two new callbacks to `gb2-terminal-expo/hooks/useStripeCallbacks.js` in the `useStripeTerminal` configuration (around line 136, after `onDidChangeOfflineStatus`):

```javascript
onDidForwardPaymentIntent: async (paymentIntent) => {
    await logger.info("✅ Payment forwarded successfully", {
        paymentIntentId: paymentIntent.id,
        amount: paymentIntent.amount,
        status: paymentIntent.status
    });
    postWebMessage(null, "goodbricks.paymentForwardedSuccess", paymentIntent);
},

onDidFailForwardingPaymentIntent: async (error) => {
    await logger.error("❌ Payment forwarding FAILED", {
        paymentIntentId: error.paymentIntentId,
        errorCode: error.code,
        errorMessage: error.message
    });
    
    // Send error to web layer for user notification
    postWebMessage(null, "goodbricks.paymentForwardedFailed", {
        paymentIntentId: error.paymentIntentId,
        error: error,
        actionDisplayName: "Offline Payment Sync Failed"
    });
},
```

---

## Next Steps: Implementation Options

### Option 1: Silent Logging (Minimal Implementation)

**Effort:** Low (1-2 hours)  
**User Impact:** None (silent failure)

**What to do:**
- Add the two callbacks above
- Log successful and failed forwards
- Keep showing the pending count
- Let Stripe retry automatically (if it does)

**Pros:**
- Quick to implement
- No UI changes needed
- Maintains current user experience

**Cons:**
- User has no visibility into failures
- No manual intervention possible
- Failed payments may be lost

---

### Option 2: User Notification (Recommended)

**Effort:** Medium (4-6 hours)  
**User Impact:** High (transparency and control)

**What to do:**

1. **Add callbacks** (as shown above)

2. **Create web layer handler** in `gb2-terminal-web/src/store/goodbricksTerminalStore.ts`:
   ```typescript
   // Add to store state
   failedOfflinePayments: [] as Array<{
     paymentIntentId: string;
     error: any;
     timestamp: number;
   }>,
   
   // Add action
   addFailedOfflinePayment: (payment) => set((state) => ({
     failedOfflinePayments: [...state.failedOfflinePayments, payment]
   })),
   ```

3. **Add toast notification** when payment fails:
   ```typescript
   // In message handler for "goodbricks.paymentForwardedFailed"
   toast.error("⚠️ 1 offline payment failed to sync", {
     action: {
       label: "View Details",
       onClick: () => openFailedPaymentsModal()
     }
   });
   ```

4. **Create "Failed Payments" modal** showing:
   - Payment Intent ID
   - Amount
   - Error message
   - Timestamp
   - "Retry" button (if possible)
   - "Contact Support" button

5. **Add indicator to TerminalHealthStatus**:
   ```
   🟢 v1.93.0 | Terminal online | P0 | ⚠️ 2 failed | 📶 🔵
   ```

**Pros:**
- Full transparency for users
- Ability to take action on failures
- Better debugging and support
- Builds trust with users

**Cons:**
- More UI work required
- Need to design error handling UX
- May alarm users unnecessarily

---

### Option 3: Automatic Retry with Notification

**Effort:** High (8-12 hours)  
**User Impact:** High (best user experience)

**What to do:**

1. **Implement Option 2** (all notification features)

2. **Add retry logic** in native layer:
   ```javascript
   // Track failed payments with retry count
   const failedPayments = new Map();
   
   onDidFailForwardingPaymentIntent: async (error) => {
       const retryCount = failedPayments.get(error.paymentIntentId) || 0;
       
       if (retryCount < 3) {
           // Exponential backoff: 5s, 15s, 45s
           const delay = 5000 * Math.pow(3, retryCount);
           
           await logger.info(`🔄 Retrying payment forward in ${delay}ms`, {
               paymentIntentId: error.paymentIntentId,
               attempt: retryCount + 1
           });
           
           setTimeout(async () => {
               failedPayments.set(error.paymentIntentId, retryCount + 1);
               // Trigger retry (if Stripe SDK provides a method)
               // Otherwise, wait for next automatic retry
           }, delay);
       } else {
           // Max retries exceeded - notify user
           await logger.error("❌ Payment forwarding failed after 3 retries", {
               paymentIntentId: error.paymentIntentId
           });
           postWebMessage(null, "goodbricks.paymentForwardedFailed", error);
       }
   },
   ```

3. **Show retry progress in UI**:
   ```
   🔄 v1.93.0 | Terminal online | P1 (retrying...) | 📶 🔵
   ```

4. **Create "Failed Payments" management screen**:
   - List of all failed payments
   - Retry status and history
   - Manual retry button
   - Export to CSV for accounting
   - Contact support integration

**Pros:**
- Best user experience
- Automatic recovery from transient failures
- Full visibility and control
- Professional error handling

**Cons:**
- Most complex to implement
- Need to handle retry state management
- May need backend support for tracking
- Requires thorough testing

---

## Recommended Approach

**Start with Option 2 (User Notification)**, then iterate to Option 3 if needed.

### Phase 1: Basic Notification (Week 1)
1. Add callbacks to track success/failure
2. Add toast notifications for failures
3. Store failed payments in local state
4. Add failed payment count to health status badge

### Phase 2: Failed Payments UI (Week 2)
1. Create modal to view failed payments
2. Add payment details and error messages
3. Implement "Contact Support" flow
4. Add export functionality

### Phase 3: Automatic Retry (Week 3-4)
1. Implement exponential backoff retry logic
2. Add retry progress indicators
3. Create comprehensive failed payments screen
4. Add backend tracking and alerting

---

## Technical Considerations

### Data Persistence
- **Question:** Should failed payments persist across app restarts?
- **Recommendation:** Yes, store in AsyncStorage or local database
- **Implementation:** Use Zustand persist middleware or React Native AsyncStorage

### Backend Integration
- **Question:** Should failed payments be reported to backend?
- **Recommendation:** Yes, for accounting and support purposes
- **Implementation:** Send webhook to backend when payment fails after max retries

### Stripe SDK Limitations
- **Question:** Does Stripe SDK provide a manual retry method?
- **Investigation Needed:** Check Stripe Terminal SDK documentation
- **Fallback:** If no retry method, rely on SDK's automatic retry mechanism

### Payment Intent Expiration
- **Issue:** Payment intents expire after 60 minutes
- **Consideration:** Failed payments may be unrecoverable if expired
- **Solution:** Show clear message: "Payment expired - contact support for manual processing"

---

## Testing Scenarios

### Test Case 1: Network Failure During Sync
1. Make 3 offline payments
2. Go online
3. Disconnect network during sync
4. Verify error callback is triggered
5. Verify user sees notification

### Test Case 2: Expired Payment Intent
1. Make offline payment
2. Wait 60+ minutes
3. Go online
4. Verify expiration error is handled
5. Verify clear error message to user

### Test Case 3: Multiple Failed Payments
1. Make 5 offline payments
2. Simulate various failure scenarios
3. Verify all failures are tracked
4. Verify UI shows correct count
5. Verify user can view all failed payments

### Test Case 4: Retry Success
1. Make offline payment
2. Simulate transient network failure
3. Verify automatic retry
4. Verify success after retry
5. Verify UI updates correctly

---

## Files to Modify

### Native Layer (React Native)
- **`gb2-terminal-expo/hooks/useStripeCallbacks.js`**
  - Add `onDidForwardPaymentIntent` callback
  - Add `onDidFailForwardingPaymentIntent` callback

### Web Layer (React/TypeScript)
- **`gb2-terminal-web/src/store/goodbricksTerminalStore.ts`**
  - Add failed payments state
  - Add actions to track failed payments
  
- **`gb2-terminal-web/src/components/TerminalHealthStatus.tsx`**
  - Add failed payment count indicator
  - Add click handler to open failed payments modal

- **`gb2-terminal-web/src/components/FailedOfflinePaymentsModal.tsx`** (new file)
  - Create modal component
  - Display failed payment details
  - Add retry/support actions

- **`gb2-terminal-web/src/utils/postMessageHandler.ts`**
  - Add handler for `goodbricks.paymentForwardedSuccess`
  - Add handler for `goodbricks.paymentForwardedFailed`

---

## Success Metrics

### Key Performance Indicators (KPIs)

1. **Sync Success Rate**
   - Target: >99% of offline payments sync successfully
   - Measure: (successful forwards / total offline payments) × 100

2. **Time to Sync**
   - Target: <30 seconds from going online to all payments synced
   - Measure: Timestamp difference between online status and last forward

3. **Retry Success Rate**
   - Target: >80% of failed payments succeed after retry
   - Measure: (successful retries / total failures) × 100

4. **User Awareness**
   - Target: 100% of failures result in user notification
   - Measure: (notifications shown / total failures) × 100

5. **Resolution Time**
   - Target: <5 minutes from failure to resolution (retry or support contact)
   - Measure: Timestamp difference between failure and resolution

---

## Questions to Answer

1. **Does Stripe SDK automatically retry failed forwards?**
   - Need to check Stripe Terminal SDK documentation
   - If yes, how many times and with what backoff?

2. **What error codes can we expect from `didFailForwardingPaymentIntent`?**
   - Need to test various failure scenarios
   - Document all possible error codes

3. **Can we manually trigger a forward retry?**
   - Check if SDK provides a method to retry forwarding
   - If not, what's the fallback strategy?

4. **Should we store failed payments in backend database?**
   - For accounting purposes?
   - For support/debugging?
   - For analytics?

5. **What's the user's expectation when a payment fails to sync?**
   - Should they contact the customer?
   - Should they retry the payment?
   - Should they contact support?

---

## Related Documentation

- **PAYMENT_INTENT_METADATA.md** - Payment intent metadata structure
- **NETWORK_BLIP_HANDLING.md** - Network recovery during payment flow
- **RECOVERY_AND_POLLING_MECHANISM.md** - Terminal recovery system
- **logs/stripe-sample-code-offline** - Stripe offline payment sample code

---

## Contact & Support

For questions or implementation help:
- Review Stripe Terminal SDK documentation
- Check Stripe support for offline payment best practices
- Test thoroughly in staging environment before production deployment

