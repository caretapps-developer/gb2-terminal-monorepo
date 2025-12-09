# Tap to Pay on iPhone - Merchant ID Setup Guide

For **Tap to Pay on iPhone**, the Merchant ID is **not configured in the App ID capabilities page**.
You set it in **Wallet → Apple Pay** section of the Developer portal, and then reference it inside your **Xcode project**.

Here's the exact path:

---

## ✅ Where to Create / Set Your Merchant ID

### **1. Apple Developer Portal → Certificates, Identifiers & Profiles**

Go to:

**Certificates, Identifiers & Profiles → Identifiers → Merchant IDs**

Direct URL (for convenience):
https://developer.apple.com/account/resources/identifiers/merchant

### **2. Create a Merchant ID**

Click **+ → Merchant IDs → Continue → Register**.

It usually looks like:

```
merchant.com.yourcompany.yourapp
```

---

## ✅ Add Merchant ID to Your App ID

Go to:

**Identifiers → App IDs → select your App → Apple Pay**

1. Scroll to **Apple Pay**
2. Enable **Apple Pay**
3. Select your Merchant ID from the list
4. Save

⚠️ Note:
**Tap to Pay capability does not list merchant IDs** — the merchant setup lives under **Apple Pay**.

---

## ✅ Add it in Xcode

After refreshing provisioning profiles:

1. Click your Xcode project
2. Select your app target
3. Go to **Signing & Capabilities**
4. Add capability → **Apple Pay**
5. Choose the same Merchant ID

Your entitlements file will automatically include:

```xml
<key>com.apple.developer.in-app-payments</key>
<array>
    <string>merchant.com.yourcompany.yourapp</string>
</array>
```

---

## ⚠️ Important: Tap to Pay ≠ Apple Pay capability

Even though you're using **Tap to Pay**, your merchant account still lives under **Apple Pay merchant IDs**.
Tap to Pay just adds terminal functionality — it doesn't replace the wallet merchant setup.

---

## Next Steps

If you want, I can verify your screenshots or walk you through:

* Stripe / Adyen Tap to Pay setup
* Entitlements file
* Provisioning profile refresh steps

