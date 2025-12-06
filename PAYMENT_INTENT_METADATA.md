# Payment Intent Metadata Documentation

## Overview

All payment intents created by the Goodbricks Terminal include comprehensive metadata for tracking, analytics, and troubleshooting. This document describes the metadata fields added to every payment intent.

---

## Metadata Fields

### Business/Transaction Metadata
These fields are passed from the web layer and contain business-specific information:

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `slug` | string | Category slug identifier | `"general-donation"` |
| `total_amount` | string | Total amount in dollars | `"100.00"` |
| `covered_fees` | string | Whether processing fees are covered | `"true"` or `"false"` |
| `covered_fees_amount` | string | Amount of covered fees | `"3.50"` |
| `donation_date` | string | Human-readable donation date | `"12/2/2025, 10:30:45 AM"` |
| `donation_date_epoch` | string | Unix timestamp (milliseconds) | `"1733155845000"` |
| `platform` | string | Platform name | `"Goodbricks"` |
| `payment_mode` | string | Payment mode | `"Terminal"` |
| `purpose` | string | Category name | `"General Donation"` |
| `purpose_slug` | string | Category slug | `"general-donation"` |
| `reader_serial_number` | string | Reader serial number | `"STRM123456"` |
| `salesforce_id` | string | Salesforce category ID | `"a1B2c3D4e5F6"` |
| `notes` | string | Additional notes | `"Tap-to-Pay"` |
| `terminal_name` | string | Terminal name or reader serial | `"Lobby Kiosk"` |
| `location_latitude` | string | GPS latitude | `"37.7749"` |
| `location_longitude` | string | GPS longitude | `"-122.4194"` |

### Offline Mode Metadata (NEW)
These fields are added by the native layer to track offline mode configuration:

| Field | Type | Description | Example Values |
|-------|------|-------------|----------------|
| `payment_mode_type` | string | **Simple indicator: offline or online payment** | `"offline"`, `"online"` |
| `offline_mode_preference` | string | Offline mode preference from web layer | `"auto"`, `"enabled"`, `"disabled"` |
| `offline_behavior` | string | Actual Stripe offline behavior used | `"force_offline"`, `"require_online"` |
| `offline_fallback` | string | Whether fallback occurred | `"true"` (only present if fallback happened) |
| `network_status_at_creation` | string | Network status when PI was created | `"online"`, `"offline"` |
| `network_type_at_creation` | string | Network type when PI was created | `"wifi"`, `"cellular"`, `"unknown"` |

---

## Offline Mode Metadata Details

### `payment_mode_type` ⭐ **SIMPLE INDICATOR**
**Source:** Native layer (derived from `offline_behavior`)
**Purpose:** Simple, easy-to-query indicator of whether this is an offline or online payment

**Values:**
- `"offline"` - Payment intent created with offline capability (`force_offline`)
- `"online"` - Payment intent requires online connection (`require_online`)

**Logic:**
```javascript
const paymentModeType = offlineBehavior === 'force_offline' ? 'offline' : 'online';
```

**Use Cases:**
- **Quick Analytics:** Simple query to count offline vs online payments
- **Dashboard Metrics:** Easy to display "X% of payments are offline-capable"
- **Filtering:** Simple filter for reports and analytics
- **Business Intelligence:** Non-technical stakeholders can easily understand

**Example Query:**
```sql
SELECT
  payment_mode_type,
  COUNT(*) as count,
  SUM(amount) as total_amount
FROM payment_intents
GROUP BY payment_mode_type;
```

### `offline_mode_preference`
**Source:** Web layer (passed via `offlineModePreference` parameter)  
**Purpose:** Indicates the user's offline mode preference setting

**Values:**
- `"auto"` - Default setting, enables offline mode automatically
- `"enabled"` - Offline mode explicitly enabled
- `"disabled"` - Offline mode explicitly disabled

### `offline_behavior`
**Source:** Native layer (determined based on `offline_mode_preference`)  
**Purpose:** The actual Stripe SDK offline behavior parameter used

**Values:**
- `"force_offline"` - Payment intent created with offline capability (used when preference is `"auto"` or `"enabled"`)
- `"require_online"` - Payment intent requires online connection (used when preference is `"disabled"`)

**Logic:**
```javascript
const shouldEnableOfflineMode = offlineModePreference !== 'disabled';
let offlineBehavior = shouldEnableOfflineMode ? 'force_offline' : 'require_online';
```

### `offline_fallback`
**Source:** Native layer (only added when fallback occurs)  
**Purpose:** Indicates that the reader doesn't support offline mode and we fell back to `require_online`

**Values:**
- `"true"` - Only present when fallback occurred
- Not present - No fallback occurred

**When it happens:**
If the reader doesn't support offline mode, the Stripe SDK returns error code `OfflineBehaviorForceOfflineWithFeatureDisabled`. The system automatically retries with `require_online` and adds this flag.

### `network_status_at_creation`
**Source:** Native layer (from `netInfoState.isConnected`)  
**Purpose:** Indicates whether the device had internet connectivity when the payment intent was created

**Values:**
- `"online"` - Device was connected to internet
- `"offline"` - Device was not connected to internet

**Use Cases:**
- Analytics: Track how many payment intents are created while offline
- Troubleshooting: Understand if network issues occurred during creation
- Correlation: Compare with `offline_behavior` to see if settings match reality

### `network_type_at_creation`
**Source:** Native layer (from `netInfoState.type`)  
**Purpose:** Indicates the type of network connection when the payment intent was created

**Values:**
- `"wifi"` - Connected via WiFi
- `"cellular"` - Connected via cellular data
- `"ethernet"` - Connected via ethernet (rare on mobile)
- `"unknown"` - Network type could not be determined
- `"none"` - No network connection

**Use Cases:**
- Analytics: Track which network types are most reliable
- Troubleshooting: Identify if certain network types cause issues
- Performance: Analyze payment success rates by network type

---

## Example Metadata

### Example 1: Online Payment with Offline Mode Enabled (Tap-to-Pay)

```json
{
  "slug": "general-donation",
  "total_amount": "50.00",
  "covered_fees": "false",
  "covered_fees_amount": "0",
  "donation_date": "12/2/2025, 10:30:45 AM",
  "donation_date_epoch": "1733155845000",
  "platform": "Goodbricks",
  "payment_mode": "Terminal",
  "purpose": "General Donation",
  "purpose_slug": "general-donation",
  "reader_serial_number": "STRM123456",
  "salesforce_id": "a1B2c3D4e5F6",
  "notes": "Tap-to-Pay",
  "terminal_name": "Lobby Kiosk",
  "location_latitude": "37.7749",
  "location_longitude": "-122.4194",
  "payment_mode_type": "offline",
  "offline_mode_preference": "auto",
  "offline_behavior": "force_offline",
  "network_status_at_creation": "online",
  "network_type_at_creation": "wifi"
}
```

### Example 2: Offline Payment (Network Dropped Before Creation)

```json
{
  "slug": "building-fund",
  "total_amount": "100.00",
  "covered_fees": "true",
  "covered_fees_amount": "3.50",
  "donation_date": "12/2/2025, 11:15:30 AM",
  "donation_date_epoch": "1733158530000",
  "platform": "Goodbricks",
  "payment_mode": "Terminal",
  "purpose": "Building Fund",
  "purpose_slug": "building-fund",
  "reader_serial_number": "STRM789012",
  "salesforce_id": "b2C3d4E5f6G7",
  "notes": "Single-Page Layout",
  "terminal_name": "Main Entrance",
  "location_latitude": "34.0522",
  "location_longitude": "-118.2437",
  "payment_mode_type": "offline",
  "offline_mode_preference": "enabled",
  "offline_behavior": "force_offline",
  "network_status_at_creation": "offline",
  "network_type_at_creation": "none"
}
```

### Example 3: Online-Only Payment (Offline Mode Disabled)

```json
{
  "slug": "special-event",
  "total_amount": "250.00",
  "covered_fees": "false",
  "covered_fees_amount": "0",
  "donation_date": "12/2/2025, 2:45:15 PM",
  "donation_date_epoch": "1733171115000",
  "platform": "Goodbricks",
  "payment_mode": "Terminal",
  "purpose": "Special Event",
  "purpose_slug": "special-event",
  "reader_serial_number": "STRM345678",
  "salesforce_id": "c3D4e5F6g7H8",
  "notes": "Multi-Page Layout",
  "terminal_name": "Event Booth",
  "location_latitude": "40.7128",
  "location_longitude": "-74.0060",
  "payment_mode_type": "online",
  "offline_mode_preference": "disabled",
  "offline_behavior": "require_online",
  "network_status_at_creation": "online",
  "network_type_at_creation": "cellular"
}
```

### Example 4: Fallback to Online (Reader Doesn't Support Offline)

```json
{
  "slug": "general-donation",
  "total_amount": "75.00",
  "covered_fees": "false",
  "covered_fees_amount": "0",
  "donation_date": "12/2/2025, 4:20:00 PM",
  "donation_date_epoch": "1733176800000",
  "platform": "Goodbricks",
  "payment_mode": "Terminal",
  "purpose": "General Donation",
  "purpose_slug": "general-donation",
  "reader_serial_number": "STRM901234",
  "salesforce_id": "d4E5f6G7h8I9",
  "notes": "Tap-to-Pay",
  "terminal_name": "Side Entrance",
  "location_latitude": "41.8781",
  "location_longitude": "-87.6298",
  "payment_mode_type": "online",
  "offline_mode_preference": "auto",
  "offline_behavior": "require_online",
  "offline_fallback": "true",
  "network_status_at_creation": "online",
  "network_type_at_creation": "wifi"
}
```

---

## Analytics Use Cases

### 1. Simple Offline vs Online Count ⭐ **MOST COMMON**
Get a quick count of offline vs online payments:
```sql
SELECT
  payment_mode_type,
  COUNT(*) as count,
  SUM(amount) as total_amount,
  AVG(amount) as avg_amount
FROM payment_intents
GROUP BY payment_mode_type;
```

**Example Result:**
| payment_mode_type | count | total_amount | avg_amount |
|-------------------|-------|--------------|------------|
| offline | 1,234 | $123,456.78 | $100.05 |
| online | 567 | $56,789.12 | $100.16 |

### 2. Offline Mode Adoption
Track how many organizations are using offline mode:
```sql
SELECT
  offline_mode_preference,
  COUNT(*) as count
FROM payment_intents
GROUP BY offline_mode_preference;
```

### 2. Network Reliability
Identify payment intents created while offline:
```sql
SELECT 
  network_status_at_creation,
  network_type_at_creation,
  COUNT(*) as count
FROM payment_intents
GROUP BY network_status_at_creation, network_type_at_creation;
```

### 3. Offline Fallback Rate
Track how often readers don't support offline mode:
```sql
SELECT 
  COUNT(*) as total_offline_attempts,
  SUM(CASE WHEN offline_fallback = 'true' THEN 1 ELSE 0 END) as fallback_count,
  (SUM(CASE WHEN offline_fallback = 'true' THEN 1 ELSE 0 END) * 100.0 / COUNT(*)) as fallback_percentage
FROM payment_intents
WHERE offline_behavior = 'require_online' AND offline_mode_preference != 'disabled';
```

### 4. Network Type Performance
Compare success rates by network type:
```sql
SELECT 
  network_type_at_creation,
  COUNT(*) as total,
  SUM(CASE WHEN status = 'succeeded' THEN 1 ELSE 0 END) as succeeded,
  (SUM(CASE WHEN status = 'succeeded' THEN 1 ELSE 0 END) * 100.0 / COUNT(*)) as success_rate
FROM payment_intents
GROUP BY network_type_at_creation;
```

---

## Troubleshooting Use Cases

### Issue: Payment Failed with "Offline Mode Not Supported"
**Check:**
1. Look at `offline_fallback` - Is it `"true"`?
2. Look at `offline_behavior` - Is it `"require_online"` when preference was `"auto"` or `"enabled"`?
3. **Diagnosis:** Reader doesn't support offline mode, system fell back to online-only

### Issue: Payment Failed with "No Internet Connection"
**Check:**
1. Look at `network_status_at_creation` - Was it `"offline"`?
2. Look at `offline_behavior` - Is it `"require_online"`?
3. **Diagnosis:** Payment intent created with online-only mode while offline

### Issue: Unexpected Offline Payment
**Check:**
1. Look at `offline_mode_preference` - Is it `"auto"` or `"enabled"`?
2. Look at `offline_behavior` - Is it `"force_offline"`?
3. Look at `network_status_at_creation` - Was it `"online"` or `"offline"`?
4. **Diagnosis:** Offline mode was enabled, network dropped during confirmation

---

## Implementation Details

### Code Location
**File:** `gb2-terminal-expo/hooks/useStripePayment.js`  
**Function:** `handleCreatePaymentIntent`  
**Lines:** 90-142

### How It Works
1. Web layer passes `offlineModePreference` via PostMessage API
2. Native layer receives preference and determines `offlineBehavior`
3. Native layer reads current network status from `netInfoState`
4. All metadata is added to the payment intent during creation
5. If offline mode fails, system retries with `require_online` and adds `offline_fallback` flag

---

## Related Documentation

- [RECOVERY_AND_POLLING_MECHANISM.md](./RECOVERY_AND_POLLING_MECHANISM.md) - Recovery system documentation
- [NETWORK_BLIP_HANDLING.md](./NETWORK_BLIP_HANDLING.md) - Network blip recovery for tap-to-pay
- [POSTMESSAGE_API.md](./POSTMESSAGE_API.md) - PostMessage API documentation

---

**Last Updated:** 2025-12-02  
**Version:** 1.89 (Web) / Build 40 (Native)  
**Author:** Goodbricks Engineering Team

