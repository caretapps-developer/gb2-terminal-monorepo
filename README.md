# GB2 Terminal Monorepo

This is a container monorepo for related GB2 Terminal projects. Each project is maintained as an independent git submodule, allowing them to be developed standalone while providing AI coding assistants with complete context across all related projects.

## Architecture Overview

This monorepo contains two tightly integrated projects that work together to provide a complete Stripe Terminal payment solution:

### 🌐 gb2-terminal-web
Web application built with React, TypeScript, Vite, and **XState** for state management.

**Location:** `./gb2-terminal-web`
**Repository:** [gb2-terminal-web](https://github.com/caretapps-developer/gb2-terminal-web.git)

**Role:** Provides the complete UI layer with event listing and user interface components. This web application is embedded as a WebView inside the Expo mobile app.

**Key Technologies:**
- React + TypeScript + Vite
- XState for state management
- PostMessage API for communication with the native app

### 📱 gb2-terminal-expo
Mobile application built with React Native and Expo.

**Location:** `./gb2-terminal-expo`
**Repository:** [gb2-terminal-expo](https://github.com/caretapps-developer/gb2-terminal-expo.git)

**Role:** Native mobile wrapper that handles all Stripe Terminal hardware communication and in-person payment processing. Embeds the web application as a WebView and communicates bidirectionally via PostMessage.

**Key Technologies:**
- React Native + Expo
- Stripe Terminal SDK (for reader communication)
- WebView with PostMessage bridge

### Communication Flow

```
┌─────────────────────────────────────────────────────────────┐
│  gb2-terminal-expo (React Native)                           │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Stripe Terminal SDK                                  │  │
│  │  • Reader connection                                  │  │
│  │  • Payment processing                                 │  │
│  │  • Hardware communication                             │  │
│  └───────────────────────────────────────────────────────┘  │
│                          ↕                                   │
│                    PostMessage API                           │
│                          ↕                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  WebView (gb2-terminal-web)                           │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  XState State Machines                          │  │  │
│  │  │  • UI State Management                          │  │  │
│  │  │  • Event Listing                                │  │  │
│  │  │  • User Interface                               │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Message Flow:**
- **Web → Native**: UI events, user actions, payment requests
- **Native → Web**: Stripe reader status, payment results, hardware events

## Features

### Payment Layouts

The terminal supports multiple payment layouts to accommodate different use cases:

#### 1. **Single Page Layout**
All-in-one screen where users select category and enter amount on the same screen.
- **Best for:** Self-service kiosks, larger screen devices
- **User flow:** Select category → Enter amount → Pay

#### 2. **Multi Page Layout**
Wizard-style flow with category selection and amount entry on separate screens.
- **Best for:** Fundraisers, smaller screen devices, guided experiences
- **User flow:** Select category → Enter amount → Pay

#### 3. **Multi Category Cart Layout** *(Coming Soon)*
Shopping cart interface where users can add multiple items from different categories.
- **Best for:** Multiple items/donations in a single transaction
- **User flow:** Add items to cart → Review → Pay

#### 4. **Tap-to-Pay Layout (Quick Payment)** ⚡ *NEW*
Zero-touch payment mode for high-volume, fixed-price scenarios.
- **Best for:** Event entry fees, fixed donations, membership payments, high-volume transactions
- **User flow:** Tap card → Payment processes automatically → Success → Ready for next payment
- **Key features:**
  - Pre-configured category and fixed amount
  - No user input required
  - Ultra-fast 2-second success screen
  - Automatically loops back to ready state
  - Perfect for contactless payments
  - Terminal-side configuration (no API changes needed)

**Tap-to-Pay Configuration:**
Administrators configure tap-to-pay mode in the Terminal Configuration Screen:
1. Select "Tap-to-Pay (Quick Payment)" layout
2. Choose category from dropdown (e.g., "General Donation")
3. Enter fixed amount (e.g., "$50.00")
4. Configuration is saved to localStorage and persists across app restarts

**Tap-to-Pay Flow:**
```
Ready Screen → Card Tap → Processing (< 2s) → Success (2s) → Loop back to Ready
```

### Additional Features

- **Reader Management:** Discover, connect, and manage Stripe card readers
- **Autonomous Recovery System:** Infinite recovery from reader disconnections, internet outages, and power cycles
- **Organization Selection:** Support for multiple organizations/businesses
- **Category Configuration:** Drag-and-drop category arrangement with show/hide options
- **Cover Fees:** Optional fee coverage for donors
- **Subscription Support:** One-time, monthly, and custom duration subscriptions
- **Customer Info Capture:** Optional email and name collection
- **Real-time Status:** Battery level, connection status, payment status indicators
- **Error Handling:** Comprehensive error handling with user-friendly messages
- **Offline Support:** Graceful handling of network disconnections

### Autonomous Recovery System

The terminal includes a comprehensive **ReaderHealthManager** component that provides infinite recovery capabilities for unattended kiosk operation:

#### Recovery Features
- **Infinite Retry** - Never gives up trying to reconnect (can run for days/weeks)
- **Exponential Backoff** - Smart retry strategy (30s → 1m → 2m → 5m ceiling)
- **Security Reboot Handling** - Automatically handles Stripe M2 reader security reboots (~13 hours)
- **Internet Outage Recovery** - Keeps trying until internet connection returns
- **Power Cycle Recovery** - Reconnects when reader is powered back on
- **Stuck Payment Detection** - Cancels payments stuck for >5 minutes
- **Payment Intent Timeout Management** - Proactive refresh at 50 minutes, critical timeout at 60 minutes
- **Software Update Detection** - Skips recovery during reader software updates

#### Health Monitoring
The system performs comprehensive health checks every 30 seconds:

| Condition | Detection | Recovery Action | Retry Interval |
|-----------|-----------|-----------------|----------------|
| **Reader disconnected** | `readerConnectionStatus !== "connected"` | Discover + reconnect | 30s → 1m → 2m → 5m (max) |
| **Security reboot** | `disconnectReason === "securityReboot"` | Wait 60s, then reconnect | 60s wait + exponential backoff |
| **Internet outage** | Network unavailable | Keep trying | Exponential backoff |
| **Stuck payment** | Payment waiting >5 min | Cancel payment intent | Immediate |
| **Payment timeout (proactive)** | Payment intent age >50 min | Cancel & recreate | Immediate |
| **Payment timeout (critical)** | Payment intent age >60 min | Force cancel | Immediate |
| **Software update** | `readerSoftwareUpdate === true` | Skip recovery | Wait for update completion |

#### Technical Details
- **Component:** `ReaderHealthManager.tsx` in `gb2-terminal-web/src/components/`
- **Polling Interval:** 30 seconds (configurable)
- **Store Integration:** Reads from `GoodbricksTerminalStore` for reader status, disconnect reason, payment intent age
- **Logging:** Detailed health metrics logged for debugging and monitoring
- **Resource Efficient:** Exponential backoff prevents system overload

#### Use Cases
Perfect for self-hosted terminals that require:
- ✅ **24/7 operation** - Kiosks, donation stations, event entry
- ✅ **Zero manual intervention** - Fully autonomous recovery
- ✅ **Resilience** - Handles internet outages, power cycles, reader reboots
- ✅ **High reliability** - Automatically recovers from any failure state

## Getting Started

### Initial Clone

When cloning this repository for the first time, you need to initialize and update the submodules:

```bash
git clone <this-repo-url>
cd gb2-terminal-monorepo
git submodule init
git submodule update
```

Or clone with submodules in one command:

```bash
git clone --recurse-submodules <this-repo-url>
```

### Updating Submodules

To pull the latest changes from all submodules:

```bash
git submodule update --remote --merge
```

To update a specific submodule:

```bash
cd gb2-terminal-web  # or gb2-terminal-expo
git pull origin main
cd ..
git add gb2-terminal-web
git commit -m "Update gb2-terminal-web submodule"
```

## Working with Submodules

### Making Changes in a Submodule

Each submodule is a standalone git repository. To make changes:

1. Navigate into the submodule directory:
   ```bash
   cd gb2-terminal-web
   ```

2. Make your changes and commit them:
   ```bash
   git add .
   git commit -m "Your commit message"
   git push origin main
   ```

3. Return to the monorepo root and update the submodule reference:
   ```bash
   cd ..
   git add gb2-terminal-web
   git commit -m "Update gb2-terminal-web submodule reference"
   git push
   ```

### Adding a New Submodule

To add another related project:

```bash
git submodule add <repository-url> <directory-name>
git commit -m "Add <project-name> submodule"
```

### Removing a Submodule

If you need to remove a submodule:

```bash
git submodule deinit -f <submodule-path>
git rm -f <submodule-path>
rm -rf .git/modules/<submodule-path>
git commit -m "Remove <submodule-name> submodule"
```

## Project Structure

```
gb2-terminal-monorepo/
├── .gitmodules              # Submodule configuration
├── .gitignore              # Root-level gitignore
├── README.md               # This file - Monorepo overview, features, and recovery system
├── POSTMESSAGE_API.md      # Complete PostMessage API contract documentation
├── BRANCHING_GUIDE.md      # Guide for working with branches across submodules
├── UI_SCREENS_GUIDE.md     # Detailed UI screens, flow, and health monitoring
├── gb2-terminal-web/       # Web application submodule
│   ├── src/
│   │   ├── components/
│   │   │   ├── ReaderHealthManager.tsx    # Autonomous recovery system
│   │   │   └── ReactNativeBridge.tsx      # PostMessage communication
│   │   └── utils/
│   │       └── GoodbricksTerminalStore.tsx # Zustand state management
└── gb2-terminal-expo/      # Mobile application submodule
    └── hooks/
        ├── useStripeReader.js              # Reader connection management
        └── useReaderSessionManager.js      # 2-hour session timeout handling
```

## Documentation

- **[README.md](./README.md)** - This file: Monorepo overview, architecture, and features
- **[POSTMESSAGE_API.md](./POSTMESSAGE_API.md)** - Complete PostMessage API contract between web and native layers
- **[BRANCHING_GUIDE.md](./BRANCHING_GUIDE.md)** - Guide for creating and managing feature branches across submodules
- **[UI_SCREENS_GUIDE.md](./UI_SCREENS_GUIDE.md)** - Comprehensive guide to all UI screens, flows, and state machine events

## Integration Details

### PostMessage Communication

The two applications communicate using the PostMessage API. See **[POSTMESSAGE_API.md](./POSTMESSAGE_API.md)** for the complete API contract, message types, and usage examples.

**Quick Overview:**

**From Web (gb2-terminal-web):**
```javascript
// Sending messages to React Native
postMessageToTerminal('discoverReaders', {});
postMessageToTerminal('createPaymentIntent', { amount, currency, ... });
```

**From Native (gb2-terminal-expo):**
```javascript
// Sending messages to WebView
postWebMessage(executionContext, "goodbricks.discoveredReaders", readers);
postWebMessage(executionContext, "goodbricks.changePaymentStatus", "ready");
```

Key message flows:
- **Web → Native**: `discoverReaders`, `connectReader`, `createPaymentIntent`, `disconnectReader`
- **Native → Web**: `goodbricks.discoveredReaders`, `goodbricks.readerConnected`, `goodbricks.changePaymentStatus`, `goodbricks.paymentCaptured`

### XState Integration

The web application uses XState for managing complex UI states and coordinating with the native layer. State machines handle:
- Payment flow states
- Reader connection states
- Event listing and updates
- Error handling and recovery

### Development Considerations

When working across both projects:

1. **Message Contract**: Ensure message types and payloads are consistent between web and native
2. **State Synchronization**: XState machines in the web app should reflect the actual hardware state from native
3. **Error Handling**: Both layers need to handle communication failures gracefully
4. **Testing**: Test the PostMessage bridge thoroughly, especially edge cases
5. **Debugging**: Use React Native Debugger to inspect WebView messages

## Benefits of This Setup

1. **Standalone Projects**: Each project maintains its own git history and can be developed independently
2. **Unified Context**: AI coding assistants can access code from all related projects for better context and understanding of the PostMessage contract
3. **Version Control**: The monorepo tracks specific commits of each submodule
4. **Flexible Development**: Work on projects individually or together as needed
5. **Separation of Concerns**: UI logic (web) is separated from hardware/payment logic (native)
6. **Web Development Speed**: UI can be developed and tested in a browser before integration
7. **Cross-Platform Potential**: The web UI could potentially be reused in other contexts

## Development Workflow

### Standalone Web Development
Develop and test the UI independently in a browser:

```bash
cd gb2-terminal-web
npm install
npm run dev
```

**Note**: When developing standalone, you may need to mock the PostMessage API to simulate native responses.

### Integrated Development
Test the full integration with the native wrapper:

```bash
# Terminal 1: Build/serve the web app
cd gb2-terminal-web
npm install
npm run build  # or dev with appropriate config

# Terminal 2: Run the Expo app
cd gb2-terminal-expo
npm install
npm start
# Then press 'i' for iOS simulator or 'a' for Android emulator
```

### Testing the Integration

1. **Web-only testing**: Test UI logic and XState machines in the browser
2. **Mock native layer**: Create mock PostMessage handlers for web development
3. **Full integration**: Test with actual Stripe Terminal hardware in the Expo app
4. **Message logging**: Add logging to track PostMessage communication during development

## Notes

- Each submodule points to a specific commit in its repository
- When you pull changes in the monorepo, submodules won't automatically update
- Always commit submodule changes in the submodule repository first, then update the reference in the monorepo
- The `.idea/` directory is ignored at the root level for JetBrains IDEs

## Additional Documentation

- **[POSTMESSAGE_API.md](./POSTMESSAGE_API.md)** - Complete PostMessage API contract and message types
- **[BRANCHING_GUIDE.md](./BRANCHING_GUIDE.md)** - Guide for creating and managing branches across submodules

## Resources

- [Git Submodules Documentation](https://git-scm.com/book/en/v2/Git-Tools-Submodules)
- [Working with Submodules](https://github.blog/2016-02-01-working-with-submodules/)

