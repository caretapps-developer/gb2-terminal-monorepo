# PostMessage API Contract

This document defines the message contract between **gb2-terminal-web** (WebView) and **gb2-terminal-expo** (React Native).

## Message Format

All messages are JSON strings with the following structure:

### From Web → Native

```typescript
{
  event: string;        // Event name
  data: any;           // Event payload
  messageId: number;   // Timestamp-based unique ID
}
```

### From Native → Web

```typescript
{
  topic: string;           // Event topic (prefixed with "goodbricks.")
  message: any;           // Event payload
  executionContext: {     // Context for tracking async operations
    messageId: string;
  };
}
```

## Web → Native Events

Events sent from the web application to the React Native layer:

| Event | Data | Description |
|-------|------|-------------|
| `webViewLoaded` | - | Signals that the WebView has finished loading |
| `initializeTerminalForOrganization` | `{ organizationId, apiKey, ... }` | Initialize Stripe Terminal for an organization |
| `fetchInternetConnectionStatus` | - | Request current internet connection status |
| `fetchTerminalData` | - | Request terminal configuration data |
| `discoverReaders` | - | Start discovering nearby Stripe readers |
| `cancelDiscoverReaders` | - | Cancel ongoing reader discovery |
| `connectReader` | `{ serialNumber, offlineModePreference? }` | Connect to a specific reader with optional offline mode preference |
| `disconnectReader` | `{ suppressError?: boolean }` | Disconnect from current reader |
| `createPaymentIntent` | `{ amount, currency, ... }` | Create a payment intent |
| `cancelCollectPaymentMethod` | - | Cancel payment method collection |
| `cancelPaymentIntent` | `{ paymentIntentId }` | Cancel a payment intent |
| `createSetupIntent` | - | Create a setup intent for saving payment method |
| `cancelCollectSetupIntent` | - | Cancel setup intent collection |
| `cancelSetupIntent` | `{ setupIntentId }` | Cancel a setup intent |
| `getConnectedReader` | - | Get currently connected reader info |
| `simulateReaderUpdate` | `"required"` | Simulate a reader software update |
| `forceRollLogFile` | - | Force log file rotation |
| `log` | `{ type, title, message }` | Send log message to native logger |

## Native → Web Events

Events sent from React Native to the web application (all prefixed with `goodbricks.`):

### Reader Discovery & Connection

| Topic | Message Type | Description |
|-------|--------------|-------------|
| `goodbricks.discoveredReaders` | `Reader[]` | List of discovered readers |
| `goodbricks.canceledDiscoverReaders` | `string` | Reader discovery was canceled |
| `goodbricks.connectedReader` | `Reader` | Reader successfully connected |
| `goodbricks.readerConnected` | `Reader` | Reader connection confirmed |
| `goodbricks.readerDisconnected` | - | Reader disconnected |
| `goodbricks.disconnect` | - | Disconnect event |
| `goodbricks.startReaderReconnect` | - | Reader reconnection started |
| `goodbricks.failReaderReconnect` | - | Reader reconnection failed |
| `goodbricks.readerConnectionStatus` | `string` | Reader connection status update |

### Reader Status & Updates

| Topic | Message Type | Description |
|-------|--------------|-------------|
| `goodbricks.updateBatteryLevel` | `{ level: number }` | Battery level update (0-1) |
| `goodbricks.readerEvent` | `ReaderEvent` | General reader event |
| `goodbricks.reportAvailableUpdate` | `Update` | Software update available |
| `goodbricks.startInstallingUpdate` | `Update` | Update installation started |
| `goodbricks.finishInstallingUpdate` | `UpdateResult` | Update installation finished |
| `goodbricks.readerSoftwareUpdateProgress` | `number` | Update progress (0-1) |
| `goodbricks.changeOfflineStatus` | `OfflineStatus` | Offline mode status changed |

### Payment Processing

| Topic | Message Type | Description |
|-------|--------------|-------------|
| `goodbricks.paymentIntentCreated` | `string` | Payment intent ID created |
| `goodbricks.changePaymentStatus` | `PaymentStatus` | Payment status changed |
| `goodbricks.requestReaderInput` | `InputType` | Reader requesting input from customer |
| `goodbricks.requestReaderDisplayMessage` | `DisplayMessage` | Reader displaying message to customer |
| `goodbricks.paymentCaptured` | `string` | Payment successfully captured |
| `goodbricks.paymentForwardedSuccess` | `{ paymentIntentId, amount, status }` | Offline payment successfully synced to Stripe |
| `goodbricks.paymentForwardedFailed` | `{ paymentIntentId, error, syncAttempts, actionDisplayName }` | Offline payment failed to sync to Stripe |

### Setup Intent (Save Payment Method)

| Topic | Message Type | Description |
|-------|--------------|-------------|
| `goodbricks.setupIntentCreated` | `SetupIntent` | Setup intent created |
| `goodbricks.paymentMethodSaved` | `SetupIntent` | Payment method saved successfully |

### Terminal Data

| Topic | Message Type | Description |
|-------|--------------|-------------|
| `goodbricks.terminalData` | `TerminalData` | Terminal configuration data |

## Type Definitions

### Reader

```typescript
interface Reader {
  id: string | null;
  serialNumber: string;
  deviceType: "chipper2X" | "stripeM2" | string;
  deviceSoftwareVersion: string | null;
  batteryLevel: number | null;        // 0-1
  batteryStatus: "nominal" | "low" | "critical" | "unknown";
  isCharging: boolean | null;
  status: "online" | "offline";
  location: Location | null;
  locationId: string | null;
  locationStatus: "set" | "notSet";
  ipAddress: string | null;
  label: string | null;
  availableUpdate: Update | null;
  simulated: boolean;
}
```

### Location

```typescript
interface Location {
  id: string;
  displayName: string;
  livemode: boolean;
  address: {
    city: string;
    country: string;
    line1: string;
    line2: string;
    postalCode: string;
    state: string;
  };
}
```

### PaymentStatus

```typescript
type PaymentStatus = 
  | "ready"
  | "waitingForInput"
  | "processing"
  | "readerInput"
  | "paymentCaptured"
  | "paymentMethodSaved"
  | string;
```

### Update

```typescript
interface Update {
  deviceSoftwareVersion: string;
  estimatedUpdateTime: number;
  requiredAt: number;
  components: string[];
}
```

### ConnectReaderData

```typescript
interface ConnectReaderData {
  serialNumber: string;           // Reader serial number (e.g., "STRM123456")
  offlineModePreference?: string;  // Optional: "auto" | "enabled" | "disabled"
}
```

**Offline Mode Preference:**
- `"auto"` (default): Use Stripe Dashboard configuration for offline mode
- `"enabled"`: Force offline mode ON regardless of Stripe Dashboard settings
- `"disabled"`: Force offline mode OFF regardless of Stripe Dashboard settings

When offline mode is enabled, payments can be collected during internet outages and are automatically synced when connection is restored.

## Usage Examples

### Web Application (gb2-terminal-web)

```typescript
// Send message to native
import { postMessageToTerminal } from './components/ReactNativeBridge';

// Discover readers
postMessageToTerminal('discoverReaders', {});

// Connect to a reader
postMessageToTerminal('connectReader', {
  serialNumber: 'STRM123456',
  offlineModePreference: 'auto' // Optional: 'auto' | 'enabled' | 'disabled'
});

// Create payment intent
postMessageToTerminal('createPaymentIntent', {
  amount: 1000,
  currency: 'usd',
  // ... other payment data
});
```

### React Native Application (gb2-terminal-expo)

```javascript
// Send message to web
import { useWebViewWithPostMessage } from './hooks/useWebViewWithPostMessage';

const { postWebMessage } = useWebViewWithPostMessage();

// Send discovered readers
postWebMessage(executionContext, "goodbricks.discoveredReaders", readers);

// Send payment status
postWebMessage(executionContext, "goodbricks.changePaymentStatus", "ready");

// Send reader connected
postWebMessage(executionContext, "goodbricks.readerConnected", readerData);
```

### Receiving Messages

**In Web (ReactNativeBridge.tsx):**

```typescript
useEffect(() => {
  const handleMessage = (event: MessageEvent) => {
    const webViewEvent = event.data;
    
    switch (webViewEvent.topic) {
      case "goodbricks.discoveredReaders":
        // Handle discovered readers
        break;
      case "goodbricks.changePaymentStatus":
        // Handle payment status change
        break;
      // ... other cases
    }
  };
  
  window.addEventListener('message', handleMessage);
  return () => window.removeEventListener('message', handleMessage);
}, []);
```

**In React Native (TerminalApp.js):**

```javascript
const handleOnMessage = async (event) => {
  const { data } = event.nativeEvent;
  const eventData = JSON.parse(data);
  const executionContext = { messageId: eventData.messageId };
  
  switch (eventData.event) {
    case 'discoverReaders':
      await handleDiscoverReaders(executionContext);
      break;
    case 'connectReader':
      await handleConnectReader(eventData.data, executionContext);
      break;
    // ... other cases
  }
};

<WebView
  ref={webViewRef}
  onMessage={handleOnMessage}
  // ... other props
/>
```

## Best Practices

1. **Always stringify JSON** before sending messages
2. **Include messageId** for tracking async operations
3. **Handle errors gracefully** - network issues can interrupt communication
4. **Log messages** during development for debugging
5. **Validate message structure** before processing
6. **Use TypeScript types** to ensure type safety across the bridge
7. **Test edge cases** like rapid message sending, connection loss, etc.

## State Synchronization

The web application uses **XState** to manage state machines that coordinate with the native layer:

- Payment flow states sync with `goodbricks.changePaymentStatus`
- Reader connection states sync with `goodbricks.readerConnectionStatus`
- Reader discovery states sync with `goodbricks.discoveredReaders`

Ensure XState machines reflect the actual hardware state from the native layer to maintain consistency.

