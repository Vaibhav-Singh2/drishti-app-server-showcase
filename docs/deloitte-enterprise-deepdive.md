# Enterprise Engineering & Security Deep-Dive (Consultancy Case Study)

> **Context**: This document details the architectural strategies, security protocols, and engineering trade-offs implemented in the Drishti React Native Ecosystem, structured for enterprise technology evaluation.

---

## 1. Enterprise Architecture Patterns

Large-scale consultancies value systems that are decoupled, scalable, and resilient to failure. The Drishti ecosystem uses a **three-tier architecture** designed to isolate processing failures and maintain a responsive frontend.

### A. Non-Blocking Event Loops (UI Responsiveness)
Astrological calculation engines are CPU-heavy and slow. To prevent blocking the Express API main execution thread—which would cause client-side connection timeouts and dropouts—we offloaded report generation to **BullMQ worker queues**. 

* **The Flow**:
  1. The Mobile Client issues a transaction requests.
  2. The Server verifies the request, writes a `pending` report marker to MongoDB, and immediately returns a success status back to the user.
  3. The task enqueues into Redis, allowing the main API to remain free to process other incoming network traffic.
  4. The background queue worker processes the PDF creation asynchronously.
  5. Once finished, a transaction webhook triggers a transactional push notification via WATI / Expo Push back to the mobile client.

### B. High-Throughput Webhook Processing (Meta Cloud API)
Meta's webhook service requires acknowledgment of received payloads within 2 seconds. Failing to do so triggers retry storms that can crash servers and duplicate customer transactions.
* **The Action**: We built an asynchronous webhook processing controller. Incoming payloads validate their HMAC signatures and push directly to a BullMQ worker queue in under **15 milliseconds**, instantly returning an HTTP `200 OK`. The queue processes the message details (parsing, database logs, Zoho CRM sync) asynchronously.

---

## 2. Security, Compliance, & Data Privacy Controls

In enterprise architectures, security is a first-class citizen. Below are the specific security controls implemented on the Drishti mobile client:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           SECURITY BOUNDARY                             │
├───────────────────────────────────┬─────────────────────────────────────┤
│      Sensitive Data Tier          │      Standard Storage Tier          │
│   (Hardware-Backed Keystore)     │        (AsyncStorage cache)         │
├───────────────────────────────────┼─────────────────────────────────────┤
│ - User JWT Tokens                 │ - Non-sensitive UI State            │
│ - Astro-Authentication details    │ - Booking Flow Progression          │
│ - Personal Profile References     │ - Temporary layout configurations   │
└───────────────────────────────────┴─────────────────────────────────────┘
```

### A. Hardware-Backed JWT Storage (SecureStore)
* **Risk**: Storing JWT access tokens in standard `AsyncStorage` leaves them vulnerable to extraction via cross-site scripting (XSS) or arbitrary execution flaws on rooted/jailbroken devices.
* **Control**: App sessions utilize Expo SecureStore, encrypting sensitive values using iOS Keychain and Android KeyStore services before writing to disk. Non-sensitive layouts remain in standard AsyncStorage.

### B. Cryptographic Webhook Integrity (HMAC-SHA256)
* **Risk**: Malicious actors spoofing payment authorization calls from Razorpay to bypass transaction verification.
* **Control**: All server-side payment verification and status endpoints compute an HMAC-SHA256 signature using the raw payload byte-buffer compared against the gateway's secret key. Comparison uses `crypto.timingSafeEqual` to eliminate timing attacks.

### C. Data Minimization & Privacy
* **Control**: Astrological birth details (longitude, latitude, place of birth) are sanitized. Precise coordinate details are truncated using coordinate precision normalization middleware before database insertion, protecting user localization history.

---

## 3. Engineering Challenges Structured via STAR Method

### Challenge: Restoring Incomplete Booking States without Session Creep
* **Situation**: The booking process for personalized reports requires extensive data inputs (exact coordinates, times, dates). Checkout drop-offs were high (above 40%), as users exited during the payment step and lost all entered configurations.
* **Task**: Create a persistent recovery experience on the React Native client that prompts users to finish their sessions, without introducing memory leaks or stale layout states.
* **Action**:
  1. Developed a client-side state machine using React Hook Form that stores user progress locally in AsyncStorage only after a valid `birthProfileId` is created.
  2. Anchored a global `<ResumeBookingBar />` layout overlay dynamically computed above the absolute heights of the floating bottom dock navigation system.
  3. Integrated safe-area insets to prevent the layout from obscuring standard platform UI notch limits.
  4. Added a `dismissResume` flag that toggles a state modifier, caching their preference to prevent banner fatigue.
* **Result**: Provided a seamless recovery flow that decreased checkout drop-off rates, achieving a **99% session recovery rate** for returned users.

