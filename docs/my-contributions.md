# My Contributions — Full Stack Developer

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

## Overview
I was the primary engineer responsible for designing the React Native mobile application structure and implementing its integration into the Express API server. My work covered designing mobile components, configuring navigation graphs, setting up local data stores, and handling real-time push and payment libraries.

---

## 1. React Native Mobile App Architecture (drishtiApp)

### Core Component Architecture
* **Hybrid Payment Verification Loop:** Designed the React Native checkout bridge mapping the native SDK (`react-native-razorpay` via `RNRazorpayCheckout` TurboModule check) with an automatic dynamic WebView fallback layer for sandbox environments.
* **Persistent Recovery Overlay:** Programmed the floating `ResumeBookingBar` recovery widget. Integrated the component inside the root router layouts, placing it dynamically above custom navigation bar tabs while computing safe-area margins to avoid layout overlap on notch screens.
* **Optimized Fluid UI Layouts:** Built the custom `CosmicSky` component utilizing native gradient interpolations (`expo-linear-gradient`) and SVG vectors (`react-native-svg`), rendering heavy cosmic graphics smoothly without standard image asset load latencies.

### Universal Routing & Deep Links
* Developed the `deeplink.ts` schema parsing system using `expo-linking` regex logic to map inbound marketing targets (SMS/WhatsApp) to nested views (e.g. `/report/[id]/read`).
* Integrated deep link listeners in the root `_layout.tsx` component, ensuring the app resolves routing paths during both cold app launches (`getInitialURL`) and warm app transitions.

### Client-Side State & Storage Architecture
* Built the local authentication structure, creating a custom `AuthContext` to persist sessions, wrapping JWT credentials securely in `expo-secure-store`.
* Programmed the application caching rules using `TanStack Query`, configuring queries with dynamic refresh locks to prevent repetitive server request loops.

---

## 2. Server-Side Integration (drishti-server)

### Payment & Verification Pipeline
* Coded the payment verify controllers, comparing client-reported transaction tokens against signature signatures.
* Developed the abandonment recovery cron daemon to synchronize incomplete user checkout sessions in the MongoDB store.

### API Integrations & Deployments
* Configured automated Docker containers for the micro-backend service, utilizing multi-stage docker packaging to build lightweight production instances.
* Coded GitHub Actions pipelines to run code linters and publish images directly to AWS EC2 container setups on code updates.

