# System Architecture

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

## 1. High-Level System Architecture

Drishti connects mobile client wrappers to a high-performance backend, orchestrating background processing queues and external services.

```mermaid
graph TB
    subgraph External["External Services"]
        ZOHO["Zoho India DC\nCRM + Books"]
        RZP["Razorpay Payment Gateway"]
        WATI["WATI WhatsApp API"]
        META["Meta Conversions API (CAPI)"]
    end

    subgraph MobileClient["drishtiApp (Expo SDK 54)"]
        Router["Expo Router\n(File Routing)"]
        Query["TanStack Query\n(Server State)"]
        Secure["SecureStore\n(JWT Auth)"]
        PayModule["Razorpay Native Module / WebView"]
        ResumeBar["Resume Booking Overlay"]
    end

    subgraph BackendServer["drishti-server (Bun + Express 5)"]
        Routes["API Routers"]
        Controllers["API Controllers"]
        
        subgraph BackgroundWorkers["BullMQ background Workers"]
            ReportQueue["report-queue\nCompiles PDF reports"]
            WebhookQueue["webhook-queue\nProcesses Zoho/RZP hooks"]
        end
        
        Logger["Winston logger\n(CloudWatch ingest)"]
    end

    subgraph DataCache["Data Infrastructure"]
        MongoDB["MongoDB Atlas\nPrimary database"]
        Redis["Redis\nQueue Backplane\n& Memory cache"]
    end

    %% Routing
    Router --> Query
    Query -->|HTTP requests| Routes
    Routes --> Controllers
    Controllers --> MongoDB
    Controllers --> BackgroundWorkers
    BackgroundWorkers <--> Redis
    Controllers --> External
    PayModule --> RZP
```

---

## 2. Inbound/Outbound Payment Verification Flow

Transactions follow a strict client-verification check paired with asynchronous webhooks to ensure Zoho CRM records update even on flaky mobile connections.

```mermaid
sequenceDiagram
    autonumber
    actor Customer as Astrological Client
    participant App as Mobile App (Expo)
    participant Server as Backend API (Bun)
    participant RZP as Razorpay Gateway
    participant Zoho as Zoho CRM / Books

    Customer->>App: Clicks "Pay & Order Report"
    App->>Server: POST /api/orders
    Server->>RZP: Request transaction session
    RZP-->>Server: Return orderId & key
    Server-->>App: Pass session credentials
    
    alt Native payment client resolved
        App->>App: Initialize RNRazorpayCheckout
        App->>Customer: Present native payment sheet
    else Browser fallback / dev sandbox
        App->>App: Load <RazorpayCheckout.web.tsx> WebView
        App->>Customer: Present fallback web form
    end

    Customer->>App: Complete payment validation
    App->>Server: POST /api/payments/verify (Send signature)
    
    par Webhook synchronization
        RZP->>Server: HTTP POST /api/webhook/razorpay (payment.authorized)
        Server->>Server: Validate HMAC signature check
        Server->>Zoho: Map transaction details & set Deal to "Closed Won"
    end
    
    Server-->>App: Transaction verified
    App->>Customer: Redirect to Success & Status check page
```

---

## 3. Deployment Topology

```mermaid
graph TB
    subgraph DevWorkspace["CI/CD Engine"]
        Git["Git push main / feature"]
        Lint["Eslint + Type check"]
        DockerBuild["Docker multi-stage build"]
        PushGHCR["Push to GitHub Container Registry (GHCR)"]
    end

    subgraph Cloud["AWS EC2 Host Container"]
        Nginx["Nginx reverse proxy\n(Port 80/443 SSL)"]
        
        subgraph DockerCompose["Docker compose network"]
            ServerApp["drishti-server (Bun container)"]
            RedisDB["Redis container"]
        end
    end

    subgraph Databases["Cloud Data Stores"]
        Atlas["MongoDB Atlas"]
        CloudWatch["AWS CloudWatch log streams"]
    end

    Git --> Lint --> DockerBuild --> PushGHCR
    PushGHCR -->|"SSH pull targets (selective)"| ServerApp
    Nginx --> ServerApp
    ServerApp --> RedisDB
    ServerApp --> Atlas
    ServerApp --> CloudWatch
```

