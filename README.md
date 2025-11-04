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
├── README.md               # This file
├── POSTMESSAGE_API.md      # Complete PostMessage API contract documentation
├── BRANCHING_GUIDE.md      # Guide for working with branches across submodules
├── gb2-terminal-web/       # Web application submodule
└── gb2-terminal-expo/      # Mobile application submodule
```

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

