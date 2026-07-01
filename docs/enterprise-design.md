# Enterprise Engineering & Governance (Deloitte Showcase Addendum)

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

## 1. Mobile Security & Compliance Framework (OWASP Alignment)

For enterprise-scale applications, mobile security is treated as a core requirement rather than an afterthought. The following architectural decisions were enforced on the mobile client:

### A. Secure Token Lifecycle Management (M1: Improper Credential Usage)
- **Problem:** Storing user JSON Web Tokens (JWT) or authentication state inside standard React state or unsecured storage (`AsyncStorage`) exposes sensitive tokens to local extraction.
- **Solution:** Configured `expo-secure-store` which maps credentials directly onto native hardware-backed keystores (**iOS Keychain Services** and **Android Keystore system**). Auth state is strictly hydrated from secure storage at app startup inside an asynchronous React Context loop (`AuthContext`), preventing cross-site scripting (XSS) or application memory sniffing from exposing tokens.

### B. Input & Configuration Validation (M5: Insufficient Cryptography)
- **Client Validation:** Enforced type-safe API requests using strict TypeScript types mapping astrological parameters.
- **Server Validation:** Configured `validateEnv.ts` utilising `Zod` schemas. If crucial API keys, environment endpoints, or JWT keys are missing, the server halts runtime immediately (`process.exit(1)`), preventing degraded or unauthenticated environments from running in production.

---

## 2. Design Patterns & System Resiliency

### A. The Facade & Fallback Pattern (Native Bridge Isolation)
To isolate payment complexities from the core React Native application layer, we used the **Facade Design Pattern**:
- The application core interacts solely with a single exposed interface: `RazorpayCheckout`.
- Under the hood, this facade evaluates runtime dependencies:
  - If the native bridge `react-native-razorpay` fails to mount (detected via runtime TurboModule verification checks), it seamlessly spins up the alternative browser fallback component.
  - This ensures that native device crashes or emulator limits never break the checkout experience for the user.

### B. Asynchronous Offloading & Fault-Tolerant Queues
- **Challenge:**Astrological calculation, layout rendering, and third-party CRM (Zoho) updates take time. Processing these inline with HTTP requests degrades server response latency and causes connection drops.
- **Solution:** Implemented **BullMQ** to offload long-running report compilation and sync tasks. If the Zoho CRM API undergoes regional downtime, the queue manager applies an **exponential backoff retry strategy** to recover transactions without dropping user purchases.

---

## 3. Environment & Configuration Governance

To coordinate multi-developer setups across Deloitte-style project teams, we configured environment boundaries:

```
                  ┌──────────────────────────────┐
                  │      GitHub Repository       │
                  └──────────────┬───────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            ▼ (Development)      ▼ (Staging/QA)       ▼ (Production)
     Local Simulators      EAS Build (Preview)   EAS Build (App Stores)
     - USE_MOCKS=true      - API: Staging URL    - API: Prod HTTPS URL
     - Local dev server    - EAS Profile: QA     - EAS Profile: Prod
```

- **Environment Isolation:** Kept `.env` variables excluded from code versioning. EAS builds ingest secrets at runtime using securely configured environment files in GitHub Action workflows.
- **Deterministic Dependencies:** Bundler setups strictly locked dependencies to enforce consistent compiles across development, QA testing, and production builds.

