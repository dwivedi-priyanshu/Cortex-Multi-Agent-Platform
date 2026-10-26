# CortexAI Architecture

This document describes the runtime components and the main authentication, agent, and payment flows visible in the current implementation.

## System architecture

```mermaid
flowchart LR
    Browser["React / Vite browser app"]
    Gateway["Express API gateway"]
    Auth["Auth service"]
    Chat["Chat service"]
    Agent["AI agent service<br/>LangGraph workflow"]
    Billing["Billing service"]
    Redis[("Redis<br/>sessions, memory, limits")]
    Mongo[("MongoDB<br/>users, conversations,<br/>messages, payments")]
    Firebase["Firebase Auth / Admin"]
    LLM["LLM providers<br/>Groq, Gemini, OpenRouter"]
    Tavily["Tavily Search"]
    Qdrant[("Qdrant<br/>PDF vectors")]
    S3[("Amazon S3<br/>generated artifacts")]
    Razorpay["Razorpay"]
    Pollinations["Pollinations image endpoint"]

    Browser -->|"HTTPS/HTTP + session cookie"| Gateway
    Gateway -->|"verify session"| Redis
    Gateway -->|"proxy auth routes"| Auth
    Gateway -->|"proxy protected chat routes<br/>forward x-user-id"| Chat
    Gateway -->|"proxy protected agent routes<br/>forward x-user-id"| Agent
    Gateway -->|"proxy protected billing routes<br/>forward x-user-id"| Billing

    Auth --> Firebase
    Auth --> Mongo
    Auth --> Redis
    Chat --> Mongo
    Agent -->|"message persistence / history"| Chat
    Agent -->|"chat memory / rate counters"| Redis
    Agent -->|"deduct credits"| Auth
    Agent --> LLM
    Agent --> Tavily
    Agent --> Qdrant
    Agent --> S3
    Agent --> Pollinations
    Billing --> Razorpay
    Billing --> Mongo
    Billing -->|"apply purchased plan"| Auth
```

### Component responsibilities

| Component | Responsibility |
| --- | --- |
| React/Vite frontend | Google sign-in UI, conversations, chat messages, file uploads, billing UI, and artifact links. |
| Express gateway | CORS and request logging, cookie-based session lookup, and routing to backend services. For protected routes it forwards the user ID in `x-user-id`. |
| Auth service | Firebase ID-token verification, user creation, Redis session management, credit deductions, and plan updates. |
| Chat service | MongoDB persistence for conversations and messages. |
| Agent service | Multipart upload handling, LangGraph routing, calling AI/search/retrieval tools, chat-memory caching, request limits, and artifact generation. |
| Billing service | Razorpay order creation, signature verification, payment persistence, and requesting plan updates from auth. |
| Redis | Shared session storage; agent chat-memory cache and per-agent request counters. |
| MongoDB | User, conversation, message, and payment documents. |
| Qdrant / S3 | Vector retrieval for PDF Q&A and storage of generated downloadable artifacts. |

## Agent request flow

```mermaid
flowchart TD
    Start["User submits prompt<br/>and optional PDF/image"] --> Frontend["Frontend POSTs /api/agent/chat"]
    Frontend --> Gateway["Gateway checks session in Redis"]
    Gateway -->|"authenticated x-user-id"| AgentController["Agent controller"]
    AgentController --> SaveUser["Save user message via chat service"]
    SaveUser --> InvokeGraph["Invoke LangGraph with prompt,<br/>agent, user, conversation, file"]

    InvokeGraph --> Explicit{"Agent explicitly selected?"}
    Explicit -->|"Yes"| Choose["Use selected agent"]
    Explicit -->|"No"| HasFile{"File attached?"}
    HasFile -->|"PDF"| PdfRag["PDF RAG agent"]
    HasFile -->|"Image"| ImageAnalysis["Image analysis agent"]
    HasFile -->|"No"| Router["LLM agent router"]
    Router --> Select["Select chat, search, coding,<br/>PDF generation, PPT, vision, etc."]
    Choose --> Tool["Selected LangGraph node"]
    Select --> Tool

    Tool --> Search{"Web search selected?"}
    Search -->|"Yes"| SearchAgent["Tavily search"]
    SearchAgent --> ChatAgent["Chat agent composes answer"]
    Search -->|"No"| Work["Run selected capability"]

    PdfRag --> Extract["Extract PDF text"]
    Extract --> Chunk["Split into chunks<br/>and embed"]
    Chunk --> Index["Index in Qdrant"]
    Index --> Retrieve["Retrieve relevant chunks"]
    Retrieve --> Work

    ImageAnalysis --> Model["Call configured model"]
    ChatAgent --> Model
    Work --> Model
    Model --> Artifact{"Creates a file?"}
    Artifact -->|"Yes"| S3["Upload artifact to S3<br/>and create signed URL"]
    Artifact -->|"No"| Answer["Build assistant response"]
    S3 --> Answer

    Answer --> Credits["Apply configured credit deduction"]
    Credits --> Persist["Cache conversation memory<br/>and save assistant message"]
    Persist --> Response["Return answer, images,<br/>and/or artifacts to frontend"]
```

### Notes on routing and persistence

- An explicit agent selection is used directly. In automatic mode, files are routed by MIME type before the LLM router is asked to classify a prompt.
- Search is a two-step path: search results are passed to the chat agent, which generates the answer.
- PDF upload analysis uses retrieval-augmented generation: extracted text is chunked, embedded, indexed in Qdrant, and searched for relevant context.
- The agent controller stores the user message before invoking the graph, then updates agent memory and persists the assistant response through the chat service.
- Agent request limits and chat memory use Redis. Current request limits are per-agent counters with a one-minute expiry.

## Authentication and payment flows

```mermaid
sequenceDiagram
    actor User
    participant UI as React frontend
    participant Gateway as API gateway
    participant Auth as Auth service
    participant Firebase as Firebase Auth/Admin
    participant Redis
    participant Billing as Billing service
    participant Razorpay
    participant Mongo as MongoDB

    User->>UI: Sign in with Google
    UI->>Firebase: Authenticate user
    Firebase-->>UI: Firebase ID token
    UI->>Gateway: POST /api/auth/login with token
    Gateway->>Auth: Proxy login request
    Auth->>Firebase: Verify ID token
    Firebase-->>Auth: Verified user claims
    Auth->>Mongo: Find or create user
    Auth->>Redis: Store session data and session mapping
    Auth-->>UI: User data + HTTP-only session cookie

    User->>UI: Select a paid plan
    UI->>Gateway: POST /api/billing/create
    Gateway->>Redis: Validate session
    Gateway->>Billing: Proxy request with x-user-id
    Billing->>Razorpay: Create order
    Razorpay-->>Billing: Order details
    Billing->>Mongo: Save payment as created
    Billing-->>UI: Order details
    UI->>Razorpay: Complete checkout
    Razorpay-->>UI: Payment and signature fields
    UI->>Gateway: POST /api/billing/verify
    Gateway->>Billing: Proxy protected request
    Billing->>Billing: Verify Razorpay HMAC signature
    Billing->>Mongo: Mark payment as paid
    Billing->>Auth: Apply plan and credits
    Auth->>Mongo: Update user plan and credits
    Auth->>Redis: Refresh cached session data
    Billing-->>UI: Payment verification result
```

## Service routes

The gateway exposes these application routes:

| Route | Method | Gateway protection | Destination |
| --- | --- | --- | --- |
| `/api/auth/login` | `POST` | Not session-protected | Auth |
| `/api/auth/logout` | `GET` | Not session-protected | Auth |
| `/api/me` | `GET` | Session required | Gateway |
| `/api/chat/create-conversation` | `GET` | Session required | Chat |
| `/api/chat/get-conversations` | `GET` | Session required | Chat |
| `/api/chat/update-conversation` | `POST` | Session required | Chat |
| `/api/chat/save-message` | `POST` | Session required | Chat |
| `/api/chat/get-messages/:conversationId` | `GET` | Session required | Chat |
| `/api/agent/chat` | `POST` | Session required | Agent |
| `/api/billing/create` | `POST` | Session required | Billing |
| `/api/billing/verify` | `POST` | Session required | Billing |

The agent service also calls the chat and auth services directly for message persistence, history retrieval, and credit deductions. These internal service URLs are configured with environment variables.

