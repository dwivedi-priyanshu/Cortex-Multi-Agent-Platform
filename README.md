# CortexAI

CortexAI is a full-stack AI assistant with a React chat interface and a set of backend services for conversations, authentication, billing, and AI-powered tools. Users can chat, search the web, get coding help, analyze uploaded PDFs and images, and generate downloadable PDF, presentation, or image artifacts.

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the service architecture and request-flow diagrams.

## Features

- Google sign-in through Firebase Authentication, with server-side token verification.
- Conversation and message history stored in MongoDB, with recent chat memory cached in Redis.
- AI agent orchestration with LangChain and LangGraph:
  - Conversational chat and web-grounded answers using Tavily search.
  - Coding assistance and code artifact generation.
  - PDF creation, PDF question answering (RAG), and image analysis.
  - Presentation and image generation.
- Document retrieval using PDF parsing, text chunking, Gemini embeddings, and Qdrant vector search.
- Generated files stored in Amazon S3 and returned through signed download URLs.
- Plan purchases through Razorpay, with plan and credit updates handled by the auth service.
- Per-agent request limits and credit deductions backed by the shared Redis/MongoDB services.

## Technology stack

| Area | Technologies |
| --- | --- |
| Frontend | React 19, Vite, Tailwind CSS, Redux Toolkit, Axios |
| API and services | Node.js, Express 5, JavaScript ES modules |
| AI orchestration | LangChain, LangGraph, Groq, Google Gemini, OpenRouter |
| Search and retrieval | Tavily, PDF Parse, LangChain text splitters, Gemini embeddings, Qdrant |
| Persistence and cache | MongoDB with Mongoose, Redis with ioredis |
| Authentication and billing | Firebase Authentication/Admin SDK, Razorpay |
| Artifact storage | Amazon S3 via AWS SDK |

## Repository layout

```text
.
├── frontend/                    # React/Vite application
│   ├── src/components/          # Chat, navigation, billing, and artifact UI
│   ├── src/features/            # API calls and frontend actions
│   ├── src/redux/               # User, conversation, and message state
│   └── utils/                   # Axios and Firebase client configuration
├── backend/
│   ├── gateway/                 # Public API entry point and session middleware
│   ├── services/
│   │   ├── agent/               # LangGraph workflow and AI tools
│   │   ├── auth/                # Firebase login, users, sessions, credits
│   │   ├── billing/             # Razorpay orders and payment verification
│   │   └── chat/                # Conversation and message persistence
│   ├── shared/redis/            # Shared Redis client
│   └── docker-compose.yml       # Local Redis container
└── ARCHITECTURE.md              # Architecture and flow diagrams
```

## Prerequisites

- Node.js and npm (use a Node.js version supported by the installed dependencies).
- Docker Desktop, or another Redis server.
- MongoDB connection string.
- Firebase project credentials for client sign-in and Firebase Admin token verification.
- Provider credentials/services used by the enabled features: Groq, Google Gemini, OpenRouter, Tavily, Qdrant, AWS S3, and Razorpay.

The repository contains separate Node.js packages for the frontend, gateway, and each backend service. Install dependencies in each package directory before starting its process.

## Configuration

Create local environment files for the frontend and each backend process. `dotenv` loads `.env` from the process working directory, so put each backend service's file in the directory from which that service is started. Do not commit credentials or Firebase service-account files.

### Frontend (`frontend/.env`)

| Variable | Purpose |
| --- | --- |
| `VITE_SERVER_URL` | Base URL of the gateway, for example `http://localhost:8000`. |
| `VITE_FIREBASE_API_KEY` | Firebase web app API key used by the client SDK. |

### Gateway (`backend/gateway/.env`)

| Variable | Purpose |
| --- | --- |
| `PORT` | Port used by the gateway. |
| `FRONTEND_URL` | Allowed frontend origin for credentialed CORS requests. |
| `AUTH_SERVICE` | Base URL of the auth service. |
| `CHAT_SERVICE` | Base URL of the chat service. |
| `AGENT_SERVICE` | Base URL of the agent service. |
| `BILLING_SERVICE` | Base URL of the billing service. |
| `REDIS_URL` | Redis connection URL used to read authenticated sessions. |

### Auth service (`backend/services/auth/.env`)

| Variable | Purpose |
| --- | --- |
| `PORT` | Port used by the auth service. |
| `MONGODB_URI` | MongoDB connection string. |
| `REDIS_URL` | Redis connection URL for user sessions and credit/session updates. |

The Firebase Admin SDK imports `backend/services/auth/config/serviceAccountKey.json`. Provide this service-account JSON locally at that path. Keep it out of version control.

### Chat service (`backend/services/chat/.env`)

| Variable | Purpose |
| --- | --- |
| `PORT` | Port used by the chat service. |
| `MONGODB_URI` | MongoDB connection string for conversations and messages. |

### Agent service (`backend/services/agent/.env`)

| Variable | Purpose |
| --- | --- |
| `PORT` | Port used by the agent service. |
| `MONGODB_URI` | MongoDB connection string. |
| `REDIS_URL` | Redis connection URL for chat memory and agent request limits. |
| `CHAT_SERVICE` | Base URL of the chat service for message persistence/history calls. |
| `AUTH_SERVICE` | Base URL of the auth service for credit deduction. |
| `QDRANT_URL` | Qdrant endpoint used by the vector store. |
| `AWS_REGION` | AWS region for S3. |
| `AWS_ACCESS_KEY_ID` | AWS access key used by the S3 client. |
| `AWS_SECRET_KEY` | AWS secret access key used by the S3 client. |
| `AWS_BUCKET_NAME` | Bucket used for generated files. |

Also configure the provider API credentials expected by the LangChain integrations in use: Groq, Google Gemini, OpenRouter, and Tavily.

### Billing service (`backend/services/billing/.env`)

| Variable | Purpose |
| --- | --- |
| `PORT` | Port used by the billing service. |
| `MONGODB_URI` | MongoDB connection string for payment records. |
| `RAZORPAY_KEY_ID` | Razorpay API key ID. |
| `RAZORPAY_KEY_SECRET` | Razorpay API key secret used for payment signature verification. |
| `AUTH_SERVICE` | Base URL of the auth service for applying a verified plan purchase. |

Set backend service URLs consistently across the gateway and service-to-service settings. Ports are supplied through `PORT`; the gateway and service URLs do not have fallback defaults in the application.

## Local development

Start Redis from the backend directory:

```powershell
cd backend
docker compose up -d redis
```

Install dependencies once in each package directory:

```powershell
cd backend
npm install

cd gateway
npm install

cd ..\services\auth
npm install

cd ..\chat
npm install

cd ..\agent
npm install

cd ..\billing
npm install

cd ..\..\..\frontend
npm install
```

Run each process in its own terminal from the corresponding package directory. Backend packages provide `npm run dev` and `npm start`; the frontend provides `npm run dev`.

After configuration and dependency installation, run `npm run dev` in separate terminals from each of these directories:

```text
backend/services/auth
backend/services/chat
backend/services/agent
backend/services/billing
backend/gateway
frontend
```

Make sure `VITE_SERVER_URL` points to the gateway and that the gateway's service URL variables point to the matching backend ports.

The Docker Compose file currently starts **Redis only**; the application services and MongoDB, Qdrant, and other external integrations must be run/configured separately.

## API overview

All browser-facing routes are served through the gateway. Protected routes require the session cookie established by login. The gateway resolves the Redis session and forwards the authenticated user ID to downstream services as `x-user-id`.

| Gateway route | Method | Description |
| --- | --- | --- |
| `/api/auth/login` | `POST` | Verify a Firebase ID token and establish a session. |
| `/api/auth/logout` | `GET` | Clear the current session. |
| `/api/me` | `GET` | Return the current authenticated user. |
| `/api/chat/create-conversation` | `GET` | Create a conversation. |
| `/api/chat/get-conversations` | `GET` | List the current user's conversations. |
| `/api/chat/update-conversation` | `POST` | Update a conversation title. |
| `/api/chat/save-message` | `POST` | Persist a message (also used internally by the agent service). |
| `/api/chat/get-messages/:conversationId` | `GET` | Load messages for a conversation. |
| `/api/agent/chat` | `POST` | Submit a prompt, optional agent selection, and optional file upload. |
| `/api/billing/create` | `POST` | Create a Razorpay order for a plan. |
| `/api/billing/verify` | `POST` | Verify a Razorpay payment and apply the purchased plan. |

The agent chat endpoint accepts multipart form data when uploading a PDF or image. The agent service currently filters uploads to PDFs and images and limits file size to 20 MB.

## Agent behavior

- Choosing an explicit agent bypasses automatic intent routing.
- In automatic mode, the agent router chooses a tool based on the prompt. Uploaded PDFs are routed to PDF question answering; uploaded images are routed to image analysis.
- Web search results flow into the chat agent to compose the user-facing answer.
- PDF question answering extracts text, splits it into overlapping chunks, indexes the chunks in Qdrant, and retrieves up to five relevant chunks for the question.
- PDF and presentation generation return downloadable artifacts; uploaded image analysis returns a textual answer.

Current credit costs and rate limits are enforced in the backend. Costs are configured by agent in the auth service; request limits are maintained in Redis by the agent service.

## Scripts

| Package | Command | Purpose |
| --- | --- | --- |
| `frontend` | `npm run dev` | Start the Vite development server. |
| `frontend` | `npm run build` | Create a production frontend build. |
| `frontend` | `npm run lint` | Run ESLint. |
| `frontend` | `npm run preview` | Preview a production build locally. |
| Each backend service | `npm run dev` | Start the service with Nodemon. |
| Each backend service | `npm start` | Start the service with Node.js. |

## Architecture

The gateway, session model, service responsibilities, and primary interaction flows are documented in [ARCHITECTURE.md](./ARCHITECTURE.md).
