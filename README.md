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
    API --> OpenAI[OpenAI Embeddings and Generation]
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
- PdfPig
- xUnit
- Docker
- GitHub Actions

## Engineering Evidence

- Clean build with zero compiler warnings
- Eighty-five automated security, persistence, ingestion, retrieval,
  evaluation, provider, controller, agent-orchestration, structured-output,
  approval, catalog, and MCP protocol tests
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
Query observability, provider-data policy evidence and automated rollback remain
incomplete.

## Current Limitations

- The first baseline is limited to five cases over one document and exact text matching
- The initial retrieval threshold exists, but recall@K and precision@K are not
  yet measured over representative multi-document data
- The local JSON metadata registry supports only a single application instance
- Production metrics, distributed traces, query audit, token use and cost
  telemetry are not yet implemented
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

The single recommended next milestone is production AI observability and cost
control for portfolio queries: model/prompt version, latency, token use,
estimated cost, outcome and safe failure metrics without logging prompts or
document content.

## CI/CD

GitHub Actions implements reusable quality, publication and deployment gates:

- Restore dependencies
- Verify formatting, warning-free builds, 85 deterministic tests and migrations
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
