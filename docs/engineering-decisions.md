# Engineering Decisions

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

## 1. Client Architecture Decisions

### Choosing Expo SDK 54 Managed Workflow
During early discovery, we weighed the benefits of a Bare React Native workflow vs. Expo Managed Workflow.
* **Bare Workflow:** High freedom to link custom native libraries manually, but introduces complex build loops, build tool version sync bugs, and slow developer onboarding.
* **Expo Managed Workflow:** Leverages Expo EAS Build pipelines and config plugins, rendering automatic Gradle/CocoaPods setups. We selected Expo Managed Workflow because Expo SDK 54 supports custom Native modules using `expo-dev-client`. This allowed integration of the native SDK `react-native-razorpay` via config plugins, while retaining automated build tools.

### State Management: TanStack Query + Context API
We evaluated Zustand, Redux Toolkit, and React Context.
* **Redux Toolkit:** Powerful but introduces boilerplate for simple CRUD-heavy client apps.
* **Zustand:** Excellent for client-side ephemeral variables, but doesn't offer built-in API query caching out of the box.
* **TanStack Query (React Query):** Selected for global server-state management. It handles caching, polling (checking report compilation status), automatic retry loops, and refetching on window focus.
* **React Context:** Retained strictly for Authentication states (`AuthContext` guarding secure route layout gates) and UI dynamic theme routing.

---

## 2. Server Architecture Decisions

### Choosing Bun Runtime
* **Node.js:** Standard, reliable ecosystem, but requires external compiler setups (e.g. ts-node, tsx, babel) and shows slower startup and package installation times.
* **Bun:** Bundles a native fast compiler, test executor, and package manager out of the box. Bun was selected because it executes typescript files directly without compiler overhead (`bun src/server.ts`), and native package installations are significantly faster.

### Selecting BullMQ + Redis Background Queues
* **Synchronous Processing:** Simpler backend controllers, but heavy database operations (compiling complex horoscope reports via template engines) run risk of blocking the main Express request thread. This degrades HTTP routing performance and causes API request timeouts.
* **BullMQ (Redis):** Selected to offload long-running tasks. Incoming reports queue as workers asynchronously. This isolates processing bugs and handles rate-limit failures gracefully using exponential backoffs.

