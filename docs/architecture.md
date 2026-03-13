# System Architecture

## Overview

The platform follows a **three-tier layered architecture** with a hard split between the runtime node (Windows container + IIS) and the processing node (local hardware via Tailscale). The C# platform handles auth, session management, BFF proxying, job orchestration, and structured data. The Python layer handles all AI operations: embedding, retrieval, agent reasoning, and guardrails.

Note: Port numbers shown are development defaults. Production port configuration is defined separately in `docker-compose.prod.yml`.

```mermaid
graph TB
    subgraph CLIENT["Client Tier"]
        BROWSER["Browser<br>SvelteKit SPA (adapter-static)"]
    end

    subgraph RUNTIME_VM["Runtime Container (Windows + IIS)"]
        subgraph IIS["IIS"]
            STATIC["Static Files<br>(SvelteKit build)"]
            ASPNET_UI["ASP.NET Core UI<br>(BFF endpoints)"]
        end

        subgraph DOCKER_RT["Docker Engine (WSL2)"]
            AGENT["Python Agent Service<br>(FastAPI :8001)"]
            QDRANT["Qdrant<br>(:6333)"]
            REDIS["Redis<br>(:6379)"]
            EMBED["bge-m3 Service<br>(:8002)"]
        end

        subgraph CSHARP_API["ASP.NET Core API (Internal)"]
            API["Internal API<br>(:5000)"]
            MSSQL[("MSSQL")]
        end
    end

    subgraph PROCESSING_NODE["Processing Node (Local Hardware - Tailscale VPN)"]
        subgraph DOCKER_PROC["Docker Engine"]
            WORKERS["Ingestion Workers<br>(Python)"]
            EMBED_PROC["bge-m3 Service<br>(:8002)"]
        end
        MINIO[("MinIO<br>(:9000)")]
    end

    BROWSER -->|HTTPS| STATIC
    BROWSER -->|HTTPS /bff/*| ASPNET_UI
    ASPNET_UI -->|HTTP internal| AGENT
    ASPNET_UI -->|HTTP internal| API
    AGENT -->|HTTP internal| API
    AGENT -->|gRPC/HTTP| QDRANT
    AGENT -->|HTTP| EMBED
    AGENT -->|TCP| REDIS
    API --> MSSQL
    API --> REDIS
    WORKERS -->|Tailscale| API
    WORKERS -->|Tailscale| QDRANT
    WORKERS --> MINIO
    WORKERS --> EMBED_PROC
```

---

## Component Responsibilities

### Browser / SvelteKit Frontend

Built with SvelteKit using `adapter-static`. The output is a static file bundle (HTML, JS, CSS) served directly from IIS. There is no SSR; all dynamic content is fetched from BFF endpoints. This means IIS has zero Python or Node.js dependency at runtime.

Key surfaces:
- **Chat widget / full-page chat**  Huginn interface with SSE streaming
- **Compliance Dashboard** Völundur fleet status + embedded chat
- **SIB HITL Review** Source-side document viewer + extracted fields editor
- **Admin Panel** Pipeline monitor, cost analytics, AI inspection history

### ASP.NET Core UI (BFF Layer)

Sits in IIS as a reverse proxy and Backend-for-Frontend. Owns session auth. The browser authenticates once via the existing C# platform auth flow. All subsequent requests carry the session cookie.

BFF (Backend For Frontend) endpoints:

| Endpoint | Purpose |
|---|---|
| `GET /bff/auth/context` | Returns current user: role, club_id, display name |
| `POST /bff/stream/query` | SSE proxy: browser → BFF → Python agent |
| `GET /bff/members/search` | Member lookup with certification data |
| `GET /bff/fleet` | Fleet data for compliance dashboard |

The SSE proxy is the critical path for streaming: the browser opens an EventSource to `/bff/stream/query`, the BFF opens an HTTP stream to the Python agent, and tokens are forwarded as they arrive. The browser never communicates with Python directly.

### ASP.NET Core API (Internal)

The central coordination service. Not exposed to the browser. Owns:
- **Job state machine** All ingestion job state in MSSQL
- **Pull orchestration** Workers claim jobs via this API
- **Cost tracking** Token usage log, model pricing table, cost period summaries
- **Query log** Every agent query recorded with TrackingId
- **SIB verification** Approve/reject/edit extracted SIB data
- **AI inspection logging** Test pipeline results, pass rate history
- **Tenant management** Club provisioning, collection registration

### Python Agent Service (FastAPI)

The AI execution layer. Stateless by design. All persistent state lives in MSSQL, Qdrant, or Redis. Runs in Docker.

Responsibilities:
- Receive query requests from BFF (via ASP.NET UI)
- Execute five-layer guardrail pipeline on input
- Route to appropriate agent (Huginn / Nölva / Völundur)
- Execute RAG retrieval (hybrid bge-m3 via Qdrant)
- Stream response tokens via SSE
- Log query + token usage back to C# API
- Execute guardrail output validation

### Ingestion Workers (Python Processing Node)

Stateless worker processes running on local hardware (or any node). They connect to the C# API via Tailscale and pull jobs. No persistent state: job state lives in MSSQL, raw documents in MinIO, vectors in Qdrant.

Worker types: Downloader, Processor, Chunker+Embedder, Qdrant Indexer, SIB Extractor.

### Qdrant

Vector database. Multi-tenant via named collections. Stores both dense vectors (bge-m3) and sparse vectors (bge-m3 SPLADE output) per point, enabling native hybrid search. All metadata filtering happens in Qdrant, no post-retrieval filtering in Python.

### Redis

Two roles:
1. **Worker wake-up stream** C# API publishes a lightweight signal when new jobs are created; workers wake immediately rather than polling on a timer.
2. **Session chat store** `SimpleChatStore` for conversation context per session, keyed by session_id.

### MinIO

S3-compatible object storage for raw downloaded documents. Workers store raw files here after download. The Processor worker reads from MinIO. This decouples download from processing, a re-process can run against the stored raw file without re-downloading.

### MSSQL

All structured data: job state, ingestion tracking, query logs, token usage, model pricing, SIB extraction records, compliance status, dialogue flags, AI inspection run results, conversation sessions. The single source of truth for operational state.

---

## Request Flow - User Query

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant BFF as ASP.NET BFF
    participant API as C#35; Internal API
    participant Guard as Guardrails
    participant Agent as Agent (Huginn/Nölva/Völundur)
    participant RAG as RAG Layer
    participant Qdrant
    participant Claude as Claude (Anthropic)

    User->>Browser: Types query
    Browser->>BFF: POST /bff/stream/query (SSE)
    BFF->>BFF: Validate session cookie
    BFF->>API: GET /bff/auth/context → user, role, club_id
    BFF->>Agent: POST /agents/{name}/stream (HTTP stream)

    Agent->>Guard: Layer 1 - Rule-based filter (regex)
    Guard->>Guard: Layer 2 - Domain classifier (bge-m3 similarity)
    alt Confidence ambiguous
        Guard->>Claude: Layer 3 - Haiku classifier
    end
    Guard->>Agent: Input cleared

    Agent->>RAG: Route query to relevant collections
    RAG->>Qdrant: Hybrid search (dense + sparse, metadata filter)
    Qdrant-->>RAG: Top-k chunks with metadata
    RAG->>RAG: AutoMergingRetriever - promote child→parent if threshold met
    RAG-->>Agent: Retrieved context + legal reference paths

    Agent->>Claude: Synthesise response (Sonnet) with context + system prompt
    Claude-->>Agent: Complete response (buffered internally)

    Agent->>Guard: Layer 5 - Output validation (index usage, domain, tenant isolation)

    alt Validation passes
        loop Stream to browser
            Agent->>BFF: SSE token
            BFF->>Browser: SSE token
            Browser->>User: Rendered token
        end
    else Validation fails
        Agent->>API: POST /api/dialogue-flags (flag_type, detail)
        Agent->>BFF: SSE - curated error message
        BFF->>Browser: SSE - curated error message
    end

    Agent->>API: POST /api/query-log (TrackingId, tokens, citations)
    Agent->>API: POST /api/token-usage (buffered, 5-min flush)
```

---

## Data Flow - Ingestion Pipeline

```mermaid
flowchart LR
    MANIFEST["Source Manifest<br>(YAML)"]
    API["C#35; API<br>Job State Machine"]
    REDIS["Redis<br>Wake Signal"]

    subgraph DL_WORKER["Downloader Worker"]
        DL_PULL["Pull pending_download job"]
        DL_CHK["SHA-256 checksum<br>vs stored"]
        DL_FETCH["Fetch document<br>(httpx / Playwright)"]
        DL_MINIO["Store raw → MinIO"]
        DL_DONE["Report completed"]
    end

    subgraph PROC_WORKER["Processor Worker"]
        PROC_PULL["Pull pending_process job"]
        PROC_PARSE["Parse<br>(PyMuPDF / BS4)"]
        PROC_LANG["Detect language<br>(langdetect)"]
        PROC_CLEAN["Clean + normalise"]
        PROC_DONE["Report completed"]
    end

    subgraph CHUNK_WORKER["Chunker + Embedder Worker"]
        CHUNK_PULL["Pull pending_chunk job"]
        CHUNK_STRAT["Select strategy<br>by document_type"]
        CHUNK_HIER["HierarchicalNodeParser<br>(regulatory)"]
        CHUNK_SEM["SentenceSplitter<br>(other types)"]
        EMBED["Embed chunks<br>(bge-m3 dense+sparse)"]
        CHUNK_DONE["Report completed"]
    end

    subgraph IDX_WORKER["Qdrant Indexer Worker"]
        IDX_PULL["Pull pending_index job"]
        IDX_UPSERT["Upsert points to Qdrant<br>(collection + metadata payload)"]
        IDX_DONE["Report completed → active"]
    end

    subgraph SIB_BRANCH["SIB Branch (parallel)"]
        SIB_EXT["LLM Extraction<br>(Haiku - structured fields)"]
        SIB_SPANS["Source span extraction"]
        SIB_REVIEW["→ pending_sib_review<br>(HITL gate)"]
    end

    MANIFEST --> API
    API --> REDIS
    REDIS -->|wake| DL_WORKER
    DL_PULL --> DL_CHK
    DL_CHK -->|unchanged| SKIP["skipped<br>(checksum match)"]
    DL_CHK -->|changed| DL_FETCH
    DL_FETCH --> DL_MINIO --> DL_DONE
    DL_DONE -->|advance state| API
    API -->|wake| PROC_WORKER
    PROC_PULL --> PROC_PARSE --> PROC_LANG --> PROC_CLEAN --> PROC_DONE
    PROC_DONE -->|advance state| API
    API -->|wake| CHUNK_WORKER
    CHUNK_PULL --> CHUNK_STRAT
    CHUNK_STRAT --> CHUNK_HIER
    CHUNK_STRAT --> CHUNK_SEM
    CHUNK_HIER --> EMBED
    CHUNK_SEM --> EMBED
    EMBED --> CHUNK_DONE
    CHUNK_DONE -->|advance state| API
    API -->|wake| IDX_WORKER
    IDX_PULL --> IDX_UPSERT --> IDX_DONE
    IDX_DONE -->|advance state| API

    PROC_DONE -->|if SIB| SIB_BRANCH
    SIB_EXT --> SIB_SPANS --> SIB_REVIEW
```

---

## Split Deployment Topology

```mermaid
graph LR
    subgraph VM["Runtime Container (Windows, Azure West Europe)"]
        IIS_STATIC["IIS: Static SvelteKit"]
        IIS_BFF["IIS: ASP.NET BFF"]
        API_SVC["ASP.NET Internal API :5000"]
        MSSQL_VM[("MSSQL")]
        subgraph DOCKER_VM["Docker (WSL2)"]
            AGENT_SVC["Python Agent :8001"]
            QDRANT_SVC["Qdrant :6333"]
            REDIS_SVC["Redis :6379"]
            EMBED_SVC["bge-m3 :8002"]
        end
    end

    subgraph PC["Processing Node (Local Hardware)"]
        subgraph DOCKER_PC["Docker"]
            WORKERS_PC["Ingestion Workers"]
            EMBED_PC["bge-m3 :8002"]
            MINIO_PC[("MinIO :9000")]
        end
    end

    TAILSCALE["Tailscale VPN"]

    PC -->|outbound via Tailscale| TAILSCALE
    TAILSCALE -->|API + Qdrant| VM
    WORKERS_PC -->|store raw docs| MINIO_PC
```

The split works because of the pull-based architecture. Workers on the processing node make outbound HTTPS calls to the C# API via Tailscale and no inbound ports needed on the processing node. When the VM eventually handles ingestion load, the workers simply move to a Docker container on the VM with zero architecture change.

---

## Multi-Tenant Isolation

Every query and every stored vector is tenant-scoped. Isolation is enforced at two independent layers:

**Collection-level isolation (Qdrant):**

| Collection | Scope | Tenant access |
|---|---|---|
| `easa_shared` | Shared - all clubs | Always included |
| `{club_id}_national_caa` | Per-club national regulations | Own club only |
| `{club_id}_reference` | Per-club reference material | Own club only |
| `{club_id}_bylaws` | Per-club bylaws | Own club only |
| `{club_id}_training` | Per-club training material | Own club only |

**Metadata filter isolation (within collections):**

Every Qdrant point carries `club_id` in its payload. All queries include a mandatory `club_id` filter alongside the vector similarity search. A query for club A cannot retrieve vectors tagged with club B even if they share a collection.

The C# API resolves `club_id` from the authenticated session before forwarding to Python. The Python agent receives `club_id` as a parameter since it does not trust client-supplied values.

---

## Service Communication

| From | To | Protocol | Auth |
|---|---|---|---|
| Browser | IIS (static) | HTTPS | Session cookie |
| Browser | ASP.NET BFF | HTTPS | Session cookie |
| ASP.NET BFF | Python Agent | HTTP (internal Docker network) | API key (internal) |
| ASP.NET BFF | C# Internal API | HTTP (internal) | Service token |
| Python Agent | C# Internal API | HTTP | Service token |
| Python Agent | Qdrant | HTTP REST or gRPC (internal) | None (internal network) |
| Python Agent | Redis | TCP | None (internal network) |
| Python Agent | bge-m3 | HTTP | None (internal network) |
| Ingestion Workers | C# Internal API | HTTPS via Tailscale | Service token |
| Ingestion Workers | Qdrant | HTTPS via Tailscale | None |
| Ingestion Workers | MinIO | HTTP | MinIO access key |
| Ingestion Workers | Anthropic API | HTTPS | API key |
| ASP.NET BFF (SIB review) | MinIO (processing node) | HTTPS via Tailscale | MinIO access key |

---

## Agent Security

**The client sends a query string and a JWT. Nothing else is trusted.**

All context required to process an agent request (`member_id`, `club_id`), role, and permitted agent scope that is resolved by the C# BFF from the validated JWT claims. These values are never accepted from the request body. The Python agent receives server-resolved context only.

Role-based access control is enforced by the existing ASP.NET auth layer before any request reaches the agent. A user's role determines which agents they may address for example: a regular member cannot reach Völundur (MOF) regardless of what they send. If the role check fails, the request is rejected at the BFF before it is forwarded to Python.

The Python agent therefore operates in a pre-validated context: by the time a query arrives, the caller's identity, club, and permitted scope have already been established and cannot be influenced by client-supplied data.

**Tool call results are internal.** When an agent calls a C# API tool (`MemberCertificationsLookup`, `FleetRegistry`, etc.), the raw structured response stays within the Python agent. The BFF forwards only the synthesised agent response to the client, never the underlying data payloads.

**No external content is rendered.** All regulatory documents are self-hosted. Agent responses contain citations that link to source documents within the platform. No content from external URLs is rendered in the client. This eliminates the XSS attack surface from citation rendering.

**Prompt injection defence is layered.** GuardRails Layer 1 filters known injection patterns on input. The second line of defence is architectural: each agent is strictly corpus-grounded and a query attempting to extract out-of-scope data will find nothing in the collections to ground an answer with. An agent that cannot retrieve does not fabricate.
