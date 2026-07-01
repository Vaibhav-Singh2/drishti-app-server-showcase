# Challenges & Solutions

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

## Challenge 1: Resolving TurboModule Linking Race Conditions

### The Problem
Integrating the native payment module `react-native-razorpay` under React Native's New Architecture resulted in runtime errors inside development emulators (such as standard Expo Go) missing native SDK packages. The module was unregistered on the global `NativeModules` array, causing immediate crashes when users initiated report purchase wizards.

### The Solution
We implemented a dynamic runtime check utilizing React Native's global environment configurations to parse TurboModule registries before calling native libraries. If missing, the app triggers a graceful fallback loading an optimized interactive WebView checkout (`RazorpayCheckout.web.tsx`) running the standard `checkout.js` library:

```typescript
// Safe TurboModule verification for New Architecture compatibility
const turboEnabled = (global as any).__turboModuleProxy != null || (global as any).TurboModuleRegistry != null;
if (!turboEnabled && !NativeModules.RNRazorpayCheckout) {
  // Gracefully fall back to WebView Checkout flow
  onResult('error', { description: 'Razorpay native module missing. Fallback to WebView initiated.' });
  return;
}
```

---

## Challenge 2: Duplicate Analytics Tracking and Event Attribution

### The Problem
Drishti reports marketing actions using client-side SDK logs (Facebook SDK) and server-side tracking (Meta Conversions API) to ensure conversions register even when client ad-blockers interrupt network requests. However, this structure led to double-reporting of purchase events inside Meta's reporting dash, spoiling conversion accuracy data.

### The Solution
We implemented a deterministic event attribution protocol:
1. When a user enters the final purchase confirmation step, the client generates a unique transaction event token (`eventId`).
2. The client transmits the event token directly to the Facebook App SDK:
   ```typescript
   AppEventsLogger.logEvent(AppEventsLogger.AppEvents.Purchased, value, { event_id: eventId });
   ```
3. Simultaneously, the client passes this event ID to the payment database during the signature verification payload:
   ```typescript
   POST /api/payments/verify { orderId, paymentId, eventId }
   ```
4. On verification, the backend enqueues the Meta Conversions API (CAPI) background worker. CAPI transmits the identical `eventId` along with user email hashes.
5. Meta matches both event payloads using the unique `eventId` and deduplicates the transaction.

---

## Challenge 3: Incomplete Booking State Desync and Recovery UX

### The Problem
Astrological reports require lengthy form configurations (Birth date, exact coordinates, birth times). Users frequently drop off during the form filling stages or exit during payment confirmation. Users returning to the app had to re-enter all details, resulting in high churn rates.

### The Solution
We built a state-governed recovery utility (`ResumeBookingBar.tsx`) anchored above the bottom navigation tabs.
1. When a user submits birth profile configurations, a unique profile token is saved (`birthProfileId`).
2. The app stores active session progress in local AsyncStorage.
3. If the app re-opens or switches screens and finds a valid `birthProfileId` but incomplete payment status, it renders the floating recovery widget.
4. The widget reads safe area margins dynamically to prevent overlapping the navigation bar:
   ```typescript
   const insets = useSafeAreaInsets();
   const resumable = !!booking.flow && !!booking.birthProfileId;
   if (!resumable || resumeDismissed) return null;
   ```
5. Tapping the widget routes the user directly to the checkout summary page. Users can dismiss the widget, which sets a local `resumeDismissed` state flag to prevent pop-up fatigue.

