# UI Screens Guide - GB2 Terminal Web

This document explains all the screens in the gb2-terminal-web application and how they interact through the XState state machine.

## Overview

The application uses **XState** to manage screen navigation and state transitions. The main state machine is defined in `terminalMachineConfig.ts` and controls the entire user flow from initialization to payment completion.

## Screen Flow Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    INITIALIZATION PHASE                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
    1. Initialize Terminal Screen
                              ↓
    2. Organization Select Screen
                              ↓
    3. Connect Reader Screen
                              ↓
    4. Categories Configuration Screen
                              ↓
    5. Terminal Configuration Screen
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    PAYMENT PHASE                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
    6. Payment Layout Selection
       ├─ Single Page Layout (Category + Amount on one screen)
       ├─ Multi Page Layout (Category → Amount on separate screens)
       ├─ Multi Category Cart Layout (Shopping cart style)
       └─ Tap-to-Pay Layout (Fixed amount, zero-touch payments)
                              ↓
    7. Payment Processing Screen
                              ↓
    8. Customer Info Capture Screen (if needed)
                              ↓
    9. Payment Confirmation Screen
                              ↓
                    (Loop back to step 6)
```

## Detailed Screen Descriptions

### 1. Initialize Terminal Screen
**State:** `initializeTerminalScreen`  
**Component:** `InitializeTerminalScreen.tsx`

**Purpose:** First screen shown when the app loads. Initializes the terminal and checks for existing configuration.

**Actions:**
- Sends `webViewLoaded` message to native app
- Checks if terminal is already configured
- Auto-connects to previously connected reader if available

**Transitions:**
- `selectOrganization` → Organization Select Screen
- `skipToPaymentLayouts` → Payment Layouts (if already configured)

---

### 2. Organization Select Screen
**State:** `showOrganizationSelectScreen`  
**Component:** `OrganizationSelectScreen.tsx`

**Purpose:** Allows user to select which organization/business to process payments for.

**Actions:**
- Displays list of available organizations
- Sends `initializeTerminalForOrganization` to native app with selected org

**Transitions:**
- `organizationSelected` → Connect Reader Screen

---

### 3. Connect Reader Screen
**State:** `showConnectReaderScreen`  
**Component:** `ConnectReaderScreen.tsx`

**Purpose:** Discovers and connects to Stripe card readers.

**Actions:**
- Sends `discoverReaders` to native app
- Displays list of discovered readers
- Allows user to select and connect to a reader
- Shows reader status (battery, connection, etc.)

**PostMessage Events:**
- **Sends:** `discoverReaders`, `connectReader`, `cancelDiscoverReaders`
- **Receives:** `goodbricks.discoveredReaders`, `goodbricks.readerConnected`

**Transitions:**
- `readerConnected` → Categories Configuration Screen
- `back` → Organization Select Screen

**Support Articles:** 
- "Help" (ccr-help)
- "Troubleshoot" (ccr-ts)

---

### 4. Categories Configuration Screen
**State:** `showArrangeCategoriesScreen`  
**Component:** `CategoriesConfigurationScreen.tsx`

**Purpose:** Configure and arrange payment categories (e.g., "Donation", "Membership", "Event Ticket").

**Actions:**
- Display existing categories
- Allow drag-and-drop reordering
- Enable/disable categories
- Configure category settings (cover fees, etc.)

**Transitions:**
- `categoriesArranged` → Terminal Configuration Screen
- `back` → Connect Reader Screen

---

### 5. Terminal Configuration Screen
**State:** `showTerminalConfigurationScreen`
**Component:** `TerminalConfigurationScreen.tsx`

**Purpose:** Configure terminal-specific settings and choose payment layout.

**Actions:**
- Set terminal name (e.g., "Mens Prayer Area" or "Spring Gala")
- Choose layout type:
  - **Single Page Layout** - Category + Amount on one screen
  - **Multi Page Layout** - Category → Amount on separate screens
  - **Multi Category Cart Layout** - Shopping cart style (not yet implemented)
  - **Tap-to-Pay Layout** - Fixed amount, zero-touch payments
- Configure terminal metadata

**Tap-to-Pay Configuration:**
When "Tap-to-Pay (Quick Payment)" layout is selected, an additional configuration card appears:

**Configuration Options:**
- **Select Category** - Dropdown to choose from visible categories
- **Fixed Amount** - Dollar amount input field (e.g., "$50.00")
- **Validation** - Ensures both category and amount are set before proceeding
- **Error Display** - Shows inline validation errors

**Features:**
- Configuration is saved to localStorage
- Persists across app restarts
- Can be changed anytime by returning to this screen
- Only visible categories are available for selection

**Validation Rules:**
- Category must be selected
- Amount must be greater than $0
- Cannot proceed without valid configuration

**Transitions:**
- `terminalConfigured` → Payment Layouts
- `back` → Categories Configuration Screen

---

## Payment Layouts

### 6a. Single Page Layout
**State:** `paymentLayouts.singlePageLayout.showSelectCategoryAndAmountScreen`  
**Component:** `SelectCategoryAndAmountScreen.tsx`

**Purpose:** All-in-one screen where user selects category AND enters amount on the same screen.

**Features:**
- Category selection buttons
- Amount input field
- Cover fees toggle
- Subscription options (one-time, monthly, custom duration)
- Summary panel showing total
- "Pay Now" button

**Actions:**
- User selects category
- User enters amount
- User optionally enables "cover fees"
- User optionally sets up subscription
- Sends `createPaymentIntent` to native app

**Transitions:**
- `amountEntered` → Payment Processing Screen

---

### 6b. Multi Page Layout

#### Step 1: Select Category Screen
**State:** `paymentLayouts.multiPageLayout.showSelectCategoryScreen`  
**Component:** `SelectCategoryScreen.tsx`

**Purpose:** User selects a payment category.

**Features:**
- Grid of category cards
- Category images/icons
- Category descriptions

**Transitions:**
- `categorySelected` → Enter Amount Screen

#### Step 2: Enter Amount Screen
**State:** `paymentLayouts.multiPageLayout.showEnterAmountScreen`  
**Component:** `EnterAmountScreen.tsx`

**Purpose:** User enters payment amount for the selected category.

**Features:**
- Shows selected category (with option to change)
- Amount input field
- Cover fees toggle
- Subscription options
- Quick amount buttons (if configured)
- "Pay Now" button

**Auto-timeout:** Returns to category selection after 180 seconds (3 minutes) of inactivity

**Transitions:**
- `backToCategorySelected` → Select Category Screen
- `amountEntered` → Payment Processing Screen
- After 180s timeout → Select Category Screen

---

### 6c. Multi Category Cart Layout
**State:** `paymentLayouts.multiCategoryCartLayout`
**Component:** (Not yet implemented)

**Purpose:** Shopping cart style where users can add multiple items from different categories before checkout.

---

### 6d. Tap-to-Pay Layout (Quick Payment)

#### Ready to Pay Screen
**State:** `paymentLayouts.tapToPayLayout.showReadyToPayScreen`
**Component:** `tapToPayLayout/ReadyToPayScreen.tsx`

**Purpose:** Zero-touch payment mode for high-volume, fixed-price scenarios. The terminal is always ready to accept payments for a pre-configured category and amount.

**Use Cases:**
- Event entry fees (e.g., "$25 admission")
- Fixed donations (e.g., "$50 donation")
- Membership payments (e.g., "$100 annual membership")
- High-volume transactions where speed is critical

**Features:**
- **Large Amount Display** - Shows fixed amount prominently (e.g., "$50.00")
- **Category Display** - Shows category name (e.g., "General Donation")
- **Pulsing Card Animation** - Visual indicator showing NFC, insert, and swipe icons
- **Reader Status** - Shows reader connection status
- **Organization Logo** - Displays organization branding
- **Pre-created Payment Intent** - Payment intent is created immediately when screen loads
- **Auto-collect Mode** - Reader automatically waits for card tap/swipe/insert

**Configuration:**
The category and fixed amount are configured in the Terminal Configuration Screen:
- Admin selects "Tap-to-Pay (Quick Payment)" layout
- Admin chooses category from dropdown
- Admin enters fixed amount
- Configuration is saved to localStorage and persists across app restarts

**Technical Details:**
- Reads configuration from `tapToPayConfig` in Zustand store
- Pre-creates payment intent on mount with `autoCollect: true` flag
- Watches `paymentProcessingStatus` for status changes
- Automatically transitions to Quick Success Screen when payment is captured

**PostMessage Events:**
- **Sends:** `createPaymentIntent` (with autoCollect flag)
- **Receives:** `goodbricks.changePaymentStatus`, `goodbricks.paymentCaptured`

**Payment Flow:**
1. Screen loads → Pre-creates payment intent
2. Reader enters "waiting for input" mode
3. Customer taps/swipes/inserts card
4. Payment processes automatically (< 2 seconds)
5. Transitions to Quick Success Screen

**Transitions:**
- `paymentCaptured` → Quick Success Screen

**Auto-timeout:** None - screen stays active indefinitely

---

#### Quick Success Screen
**State:** `paymentFlow.showQuickSuccessScreen`
**Component:** `tapToPayLayout/QuickSuccessScreen.tsx`

**Purpose:** Brief success confirmation for tap-to-pay transactions.

**Features:**
- **Success Animation** - Green checkmark with animation
- **Amount Display** - Shows captured amount
- **Category Display** - Shows category name
- **Success Message** - "Payment Captured Successfully"
- **Organization Logo** - Displays organization branding
- **Auto-dismiss** - Automatically returns to Ready to Pay Screen after 2 seconds

**Technical Details:**
- Reads configuration from `tapToPayConfig` in Zustand store
- XState handles 2-second timeout automatically
- Clears transaction context on exit

**Transitions:**
- After 2 seconds → Ready to Pay Screen (automatic loop)

**Flow Diagram:**
```
┌─────────────────────────────────────────────────────────────┐
│                    TAP-TO-PAY FLOW                          │
└─────────────────────────────────────────────────────────────┘

    Ready to Pay Screen
    ┌─────────────────────┐
    │   $50.00            │
    │   General Donation  │
    │                     │
    │   [Card Animation]  │
    │   Tap, Insert, or   │
    │   Swipe Card        │
    └─────────────────────┘
            ↓
    Customer Taps Card
            ↓
    Processing (< 2s)
            ↓
    Quick Success Screen
    ┌─────────────────────┐
    │   ✓                 │
    │   Payment Captured  │
    │   Successfully      │
    │                     │
    │   $50.00            │
    │   General Donation  │
    └─────────────────────┘
            ↓
    Auto-dismiss (2s)
            ↓
    ┌─────────────────────┐
    │  Loop back to       │
    │  Ready to Pay       │
    └─────────────────────┘
```

**Key Advantages:**
- ⚡ **Ultra-fast** - Zero user input, 2-second success screen
- 🔄 **Continuous** - Automatically loops back to ready state
- 🎯 **Simple** - No category selection, no amount entry
- 📱 **Touch-free** - Perfect for contactless payments
- 🏃 **High-volume** - Optimized for processing many transactions quickly

---

## Payment Flow

### 7. Payment Processing Screen
**State:** `paymentFlow.showPaymentProcessingScreen`  
**Component:** `PaymentProcessingScreen.tsx`

**Purpose:** Handles the actual payment processing and shows real-time status.

**Payment Status States:**
- `initialized` - Payment intent created
- `ready` - Reader ready for card
- `waitingForInput` / `readerInput` - Waiting for customer to insert/tap card
- `processing` - Processing payment
- `removeCard` - Prompt to remove card
- `paymentCaptured` - Payment successful
- `paymentMethodSaved` - Payment method saved (for subscriptions)
- `notReady` - Reader not ready
- `paymentIntentCanceled` - Payment canceled

**Visual Feedback:**
- `PaymentProcessingAnimation` - Spinner/loading animation
- `PaymentReaderInputAnimation` - "Insert or tap card" animation
- `PaymentCaptured` - Success animation
- `PaymentMethodSaved` - Payment method saved confirmation
- `PaymentFailure` - Error state
- `PaymentReaderNotReady` - Reader connection issue

**PostMessage Events:**
- **Sends:** `createPaymentIntent`, `cancelCollectPaymentMethod`, `cancelPaymentIntent`
- **Receives:** `goodbricks.changePaymentStatus`, `goodbricks.paymentCaptured`, `goodbricks.requestReaderInput`

**Transitions:**
- `selectKeyIn` → Key-In Screen (manual card entry)
- `newCustomer` → Capture Customer Info Screen
- `confirmPayment` → Payment Confirmation Screen
- `paymentFailure` → Back to Payment Layouts
- After 180s timeout → Back to Payment Layouts

---

### 8. Capture Customer Info Screen
**State:** `paymentFlow.showCaptureCustomerInfoScreen`  
**Component:** `CaptureCustomerInfoScreen.tsx`

**Purpose:** Collect customer information for receipt and record-keeping.

**Fields:**
- Customer name
- Customer email
- Optional: Phone number, address

**Transitions:**
- `confirmPayment` → Payment Confirmation Screen

---

### 9. Payment Confirmation Screen
**State:** `paymentFlow.showPaymentConfirmationScreen`  
**Component:** `PaymentConfirmationScreen.tsx`

**Purpose:** Show payment success and offer to send receipt.

**Features:**
- Payment success message
- Transaction details (amount, category, date)
- Send receipt option
- "Done" button to start new transaction

**Actions:**
- Updates payment intent with customer info
- Sends receipt email if requested
- Resets transaction data

**Transitions:**
- `close` → Back to Payment Layouts (ready for next transaction)

---

## Alternative Payment Flow: Key-In (Manual Entry)

### Key-In Screen
**State:** `paymentFlow.showKeyInScreen`  
**Component:** `KeyInScreen.tsx`

**Purpose:** Manual card entry for situations where card reader can't be used.

**Fields:**
- Card number
- Expiration date
- CVV
- ZIP code

**Transitions:**
- `submit` → Key-In Processing Payment Screen

### Key-In Processing Payment Screen
**State:** `paymentFlow.showKeyInProcessingPaymentScreen`  
**Component:** `KeyInProcessingPaymentScreen.tsx`

**Purpose:** Process manually entered card payment.

**Transitions:**
- `keyInPaymentCaptured` → Capture Customer Info Screen
- `paymentFailure` → Back to Key-In Screen

---

## Navigation & Side Menu

The app includes a hamburger menu (accessible from all screens) with:

1. **Device Status** - Shows reader connection, battery, network status
2. **Logs** - Debug logs for troubleshooting
3. **Settings** - App configuration and preferences

---

## State Machine Events

### User-Triggered Events
- `selectOrganization` - User selects organization
- `organizationSelected` - Organization selection confirmed
- `readerConnected` - Reader successfully connected
- `categoriesArranged` - Categories configured
- `terminalConfigured` - Terminal setup complete
- `categorySelected` - Payment category chosen
- `amountEntered` - Amount entered and payment initiated
- `confirmPayment` - Payment confirmed
- `newCustomer` - New customer info needed
- `selectKeyIn` - Switch to manual card entry
- `back` - Navigate back
- `close` - Close current screen

### System-Triggered Events
- `skipToPaymentLayouts` - Skip setup (already configured)
- `paymentFailure` - Payment failed
- `keyInPaymentCaptured` - Manual payment successful
- `backToCategorySelected` - Return to category selection
- `backToArrangeCategories` - Return to category config
- `backToConnectReader` - Return to reader connection

### Layout Selection Events
- `singlePageSelected` - Use single-page layout
- `multiPageSelected` - Use multi-page layout
- `multiCategoryCartSelected` - Use cart layout
- `tapToPaySelected` - Use tap-to-pay layout

### Tap-to-Pay Events
- `cardTapped` - Card tapped/swiped/inserted (triggers payment flow)
- `paymentCaptured` - Payment successfully captured (triggers success screen)
- `resetToReady` - Reset to ready state (return to waiting for card)

---

## Context Data

The state machine maintains context data throughout the flow:

```typescript
{
  customerEmail: string;
  customerName: string;
  totalAmount: number;
  category: {
    name: string;
    coverFees: boolean;
    // ... other category properties
  };
}
```

---

## Auto-Timeouts

Several screens have automatic timeouts to reset the UI:

- **Enter Amount Screen**: 180 seconds (3 minutes) → Returns to category selection
- **Payment Flow**: 180 seconds (3 minutes) → Returns to payment layouts

This prevents the terminal from being stuck on a screen if a customer walks away.

---

## Integration with Native Layer

Throughout the flow, the web app communicates with the React Native layer via PostMessage:

**Key Integration Points:**
1. **Reader Discovery** - Native handles Bluetooth/USB communication
2. **Payment Processing** - Native uses Stripe Terminal SDK
3. **Terminal Data** - Native provides device status, location, network info
4. **Logging** - Logs are sent to native for persistence

See [POSTMESSAGE_API.md](./POSTMESSAGE_API.md) for complete message contract.

