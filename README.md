# Enterprise AI Knowledge Assistant

A production-oriented Retrieval-Augmented Generation (RAG) project designed to help developers, technical leads, and architects find trustworthy answers in internal company documentation.

> The source repository is private. Access can be provided to recruiters or technical reviewers upon request.

## Protected Live Demonstration

The Google Cloud production portfolio exposes a Keycloak-protected Swagger page
containing exactly:

```text
GET  /api/portfolio/documents
POST /api/portfolio/ask
```

Review access is privately shared and revocable. The reviewer identity has only
the `portfolio-demo` role, the API can retrieve only two deterministic fictional
Northstar Labs documents, and requests are limited to 10 per authenticated
subject every 60 seconds. Normal reader, learning and administrative APIs are
not published at the production edge.

## Business Problem

Engineering knowledge is often fragmented across architecture documents, API references, coding standards, decision records, and troubleshooting guides. Finding an accurate answer is slow, and ordinary search does not explain which source supports it.

## Product Goal

Provide concise answers grounded in authorized company documents, with inspectable citations containing the document, page, and chunk. When evidence is insufficient, the assistant should refuse to invent an answer.

## Target Users

- Developers
- Technical leads
- Software and enterprise architects

## Current Architecture

```mermaid
flowchart LR
    Client[API Client] --> Keycloak[Keycloak OIDC]
    Client --> API[ASP.NET Core API]
    Keycloak --> API
    MCPClient[MCP Client] --> MCP[Local Read-Only MCP Server]
    API --> Files[Managed PDF Storage]
    API --> Health[Local Model Health Monitor]
    API --> Router[AI Router - health aware]
    Health -.availability.-> Router
    Router --> OpenAI[OpenAI Embeddings and Generation]
    Router -->|mesh VPN| Ollama[On-Prem Ollama Chat and Embeddings]
    API --> Qdrant[Qdrant Vector Database]
    API --> Registry[Persistent Document Registry]
    API --> AppDB[PostgreSQL Users Permissions Audit]
    Keycloak --> IdentityDB[PostgreSQL Identity State]
    MCP --> Registry
```

### Ingestion

```text
Validate PDF
→ calculate content identity
→ extract page text
→ create overlapping chunks
→ generate embeddings
→ batch vectors into Qdrant
→ persist indexing status
```

### Question Answering

```text
Question
→ validate OIDC access token
→ resolve explicit document permissions
→ create question embedding
→ retrieve only authorized chunks
→ generate a grounded answer
→ return document/page/chunk citations
```

## Implemented Capabilities

- PDF upload and validation
- Page-level text extraction
- Fixed-size overlapping chunking
- OpenAI embeddings and grounded generation
- Qdrant semantic retrieval
- Source citations
- Content-based duplicate prevention
- Stable document and vector identifiers
- Restart-safe ingestion metadata
- Recoverable indexing status
- Batched vector writes
- Versioned RAG evaluation dataset
- Deterministic fact, stable-document citation, and refusal scoring
- Cost-aware evaluation CLI with targeted case execution
- End-to-end evaluation latency measurement
- Automated build and tests with GitHub Actions
- Configurable Qdrant retrieval threshold with explicit no-evidence refusal
- Bounded single-tool agent workflow using the OpenAI Responses API
- Strict read-only document-catalog tool with inspectable execution traces
- Versioned agent policy cases with deterministic structural trace scoring
- Internal agent evaluation CLI with targeted case execution
- AI request observability and cost control for every AI provider call: RAG,
  bounded agent, and structured-action planning, tagged by surface
- Configurable per-model cost estimation, latency/cost/failure/rolling
  failure-rate alert thresholds, and a bounded administrator telemetry
  read endpoint
- Provider timeout, retry, and circuit-breaker resilience on all four
  outbound AI provider clients (OpenAI chat, embeddings, Responses API,
  Qdrant), with retry counts recorded on both recovered and hard failures
- Local Ollama chat model wrapping Microsoft.Extensions.AI's `IChatClient`
  abstraction (via OllamaSharp) instead of a hand-written HTTP client,
  selected per request by an AI router alongside the cloud model, with its
  own resilience timeout budget for CPU-bound local inference; routing is
  off unless a deployment explicitly opts in
- Local embeddings and a dedicated local vector collection, so retrieval as
  well as generation can run without leaving the local machine
- Automatic failover between local and cloud: a background health monitor
  detects an unreachable local model and routes every request to the cloud
  provider, recovering on its own when the local model returns
- Confidence-based retrieval cascade that consults the cloud tier when
  local results score below a configured threshold, keeping the better of
  the two
- Outbound-only cloud-to-on-prem connector (mesh VPN), letting the deployed
  cloud application consume inference from a machine behind NAT with no
  inbound firewall rule and no public exposure of the model
- Strict JSON Schema output for a simulated high-impact document action
- Application-validated pending, approved, and rejected proposal state
- Atomic single-decision enforcement and simulation-only approval results
- Local stdio MCP server with one safe `find_documents` tool
- Shared protocol-neutral catalog query used by both agent and MCP adapters
- Portable Keycloak OIDC authentication with strict JWT validation
- EF Core and PostgreSQL identity links, document permissions, and audit events
- Administrator policies for ingestion, permissions, and governed actions
- Document scope enforced through listing, RAG, Qdrant, and agent tools
- Docker Compose topology for API, Keycloak, separate PostgreSQL databases, and
  Qdrant
- Production Nginx TLS edge with private backend networks and file-mounted
  secrets
- Controlled EF migration, Qdrant, Keycloak and fictional-corpus initialization
  jobs
- Verified backup and restore automation with checksums and public smoke tests
- Dual-platform AMD64/ARM64 GHCR image with SBOM, provenance and immutable
  digest deployment
- Protected production Swagger with a dedicated role, exact fictional scope,
  rate limiting and no administrative exposure

## Technology Stack

- C# and .NET 10 LTS
- ASP.NET Core Web API
- Keycloak and OpenID Connect
- PostgreSQL with EF Core and Npgsql
- OpenAI API
- Qdrant
- Model Context Protocol official C# SDK
- Microsoft.Extensions.Http.Resilience (timeout, retry, circuit breaker)
- Microsoft.Extensions.AI / OllamaSharp (local model abstraction)
- Ollama
- Tailscale (WireGuard mesh VPN for the cloud-to-on-prem connector)
- PdfPig
- xUnit
- Docker
- GitHub Actions

## Engineering Evidence

- Clean build with zero compiler warnings
- 151 automated security, persistence, ingestion, retrieval, evaluation,
  provider, controller, agent-orchestration, structured-output, approval,
  catalog, observability, and MCP protocol tests
- Duplicate content rejected even under another filename
- Vector identifiers cannot collide across documents
- Stable `DocumentId` prevents filename ambiguity in citation evaluation
- Five-case live OpenAI/Qdrant baseline recorded with fact, citation, refusal, and latency evidence
- Six retrieval thresholds evaluated; the current `0.20` default preserved the
  5/5 baseline at 1,926 ms average latency
- Agent tests prove strict provider mapping, bounded execution, single-call
  policy, status/name filtering, and exclusion of local paths and content hashes
- Agent policy tests prove exact tool selection and stopping expectations,
  recursive sensitive-metadata checks, unknown-tool rejection, safe tool
  failures, malformed/incomplete provider rejection, and DI composition
- A live two-turn Responses API run invoked `find_documents`, returned two
  indexed documents, and produced a completed answer with an execution trace
- Swagger verification proved a structured deletion proposal requires a
  separate decision, repeated decisions conflict, and approval leaves the
  document registry unchanged
- A real MCP client test launches the server over stdio, discovers exactly one
  read-only/non-destructive tool, calls it, rejects an invalid status, and
  verifies sensitive registry fields are absent
- Security tests prove strict issuer/audience/signature/lifetime validation,
  fail-closed endpoint policy, stable issuer/subject user links, idempotent
  grants, durable audit, empty-scope refusal, and Qdrant provider-output checks
- EF migration discovery, model comparison, and idempotent PostgreSQL SQL
  generation are verified
- Keycloak, PostgreSQL, Qdrant, API and Nginx run in the verified production
  Compose topology on Google Cloud
- Hosted quality gates verify formatting, warning-free builds, deterministic
  tests and PDFs, migrations, shell/Keycloak/Compose contracts, and AMD64/ARM64
  production images
- The protected workflow publishes an immutable GHCR digest and deploys it
  through pinned SSH only after human production approval
- Hosted trusted-TLS smoke tests prove health, the two-operation Swagger
  contract, anonymous `401`, and public-edge `404` isolation
- Every AI provider call (RAG, bounded agent, structured-action planning) is
  measured for latency, token use, estimated cost, outcome, and retry count,
  live-verified through Swagger against real OpenAI/Qdrant calls, including
  a recorded edge case where OpenAI returned a dated snapshot model name
  that the cost estimator's fallback matching still priced correctly
- Provider timeout/retry/circuit-breaker resilience is live-verified against
  a real induced Qdrant outage in both directions: a recovered transient
  failure records a successful outcome with a non-zero retry count, and a
  sustained outage records a failed outcome that also reports how many
  attempts were made before giving up
- The local Ollama chat path is live-verified end to end through Swagger
  against a real running model: a genuine grounded RAG answer with the
  provider and model correctly recorded in observability telemetry, after
  live testing itself surfaced and fixed two real defaults (a routing
  threshold too small for real retrieved-context size, and a resilience
  timeout budget too short for CPU-bound local model loading) before
  shipping
- The fully local retrieval path is live-verified: a document indexed with
  local embeddings into the local collection, then answered from a question
  embedded and searched locally, with zero cloud calls, confirmed directly
  against the vector database rather than from application logs
- Automatic failover is live-verified across the full cycle: local model
  serving, stopped mid-session, the health monitor detecting it within one
  interval, a real question answered by the cloud provider with no
  user-visible failure, then automatic recovery without a restart
- The cloud-to-on-prem connector is verified in production: the deployed
  Google Cloud host reaches an on-premises model over an outbound-only mesh
  VPN, with no inbound firewall rule and no public exposure of the model
- The retrieval cascade's confidence threshold was corrected after live
  measurement disproved the original design: an emptiness-based fallback
  could never fire, because the local embedding model scored a question
  about an entirely unrelated subject at 0.466 against the indexed corpus
- Secrets excluded from source control

## Security Approach

The product handles internal architecture, API, coding-standard, decision, and
troubleshooting documents. Keycloak authenticates users without the application
storing passwords. The API validates OIDC access tokens and owns document
authorization in PostgreSQL. Authorization is enforced before catalog, vector,
RAG, and agent access; permission changes and governed decisions are audited.
Production terminates trusted TLS at Nginx, keeps service networks private,
mounts secrets from protected host files, runs controlled migrations, and has
verified backup/restore scripts. The public portfolio adds an exact fictional
document allow-list, a dedicated low-authority role and per-subject rate limit.
Every AI provider call is measured (latency, token use, estimated cost, retry
count, and outcome, with no prompt/answer/document content recorded) and
protected by timeout/retry/circuit-breaker resilience. An on-premises model
can answer requests instead of the cloud model, reached over an
outbound-only mesh VPN that requires no inbound firewall rule and never
exposes the model publicly. That connector is a device-authenticated
boundary: prompt content crosses it protected by transport encryption, so
an on-premises model inherits the same data-sensitivity considerations as
any other provider rather than being automatically safer for being local.
Routing selects by prompt length and availability, not yet by data
sensitivity, so runtime guardrails remain a prerequisite before broader
exposure. Query audit, distributed tracing and automated rollback remain
incomplete.

## Current Limitations

- The first baseline is limited to five cases over one document and exact text matching
- The initial retrieval threshold exists, but recall@K and precision@K are not
  yet measured over representative multi-document data
- The local JSON metadata registry supports only a single application instance
- AI request telemetry (token use, cost, latency, retry count, outcome) and
  provider resilience are implemented and locally verified but not yet
  exercised against production traffic; general HTTP request metrics and
  distributed tracing remain unimplemented
- The AI router selects between local and cloud models by prompt length and
  current local availability; it does not yet consider data sensitivity or
  task complexity, and the local adapter's retry attempts are not visible
  in telemetry the way the cloud adapters' are
- The retrieval cascade's confidence threshold is configurable but not yet
  calibrated against the evaluation dataset, and it compares similarity
  scores across two embedding models whose scales are not strictly
  comparable; a reranker is the recorded principled improvement
- The deployed environment runs local chat with cloud retrieval, because
  its local vector collection is provisioned but intentionally empty;
  populating it requires indexing documents through the local tier
- The on-prem connector is a device-authenticated VPN without per-service
  access control lists, and the local model endpoint has no authentication
  of its own — acceptable for a single-owner network, insufficient for a
  shared or client network
- The agent endpoint is authenticated and document-scoped but still has one
  read-only tool, relies on stored provider response state, has only three
  non-adversarial policy cases, and does not persist its execution trace
- The approval workflow uses authenticated reviewer identity and durable
  decision audit but stores proposals only in process memory and intentionally
  has no real deletion capability
- The MCP server is local-only and OS-trusted; it reads the whole local registry
  and must not be exposed remotely
- The portfolio reviewer credential is shared and the in-memory rate limiter
  resets on API restart, so this is a bounded single-instance demonstration
- Automatic release rollback and multi-host availability are not implemented
- Backup/restore automation exists, but encrypted off-host scheduling and timed
  recovery objectives remain to be demonstrated continuously

## Roadmap

Hybrid local/cloud AI is now running in the deployed environment: chat
requests route to an on-premises model over an outbound-only mesh VPN, with
automatic failover to the cloud provider whenever that model is
unreachable. Retrieval currently uses the cloud tier in the deployed
environment, since its local collection is intentionally empty.

The next milestone packages the assembled capability — application, local
model, retrieval, cloud fallback, authentication and audit, routing, and
deployment support — as one hybrid solution, with hardware selection
deliberately deferred until a real client's scale, latency, and budget are
known. A demo-facing client interface follows, then runtime guardrails
(tool-call authorization, rate limits, PII and prompt-injection checks)
before any of these paths are exposed more broadly.

## CI/CD

GitHub Actions implements reusable quality, publication and deployment gates:

- Restore dependencies
- Verify formatting, warning-free builds, 151 deterministic tests and migrations
- Validate shell, Keycloak, Compose and deterministic fictional-PDF contracts
- Build and inspect AMD64 and ARM64 production images
- Publish SBOM/provenance with a commit tag and immutable digest
- Deploy only reviewed `main` through the protected production environment
- Require initialization, readiness and trusted-TLS smoke tests before recording
  the digest

The current single-host Google Cloud deployment remains Compose-based and
portable; Kubernetes and additional cloud-specific infrastructure are not
justified at this stage.

## Repository Access

This public repository is a portfolio case study. The private implementation can be shared with verified recruiters or technical interviewers upon request.

## Author

**Elie Saber**  
Senior .NET Engineer and Technical Lead developing production AI engineering and Enterprise AI Architecture capability.
