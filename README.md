
# Lexora AI — Local LLM Assistant

## Purpose:
Lexora AI is an on-prem / internal assistant for engineers (developers, DevOps, application support).
It combines a Python ingestion pipeline (documents, code, configs → embeddings → Qdrant) with a Spring Boot orchestration service that serves two retrievers:

DocsRetriever — used by the Teams bot (docs & runbooks only), and

FullRetriever — used by the Streamlit web UI (docs + code + infra + logs).

This README is intentionally comprehensive: it covers architecture, exact setup steps, configuration, security, ingestion strategy, API contracts, prompt templates, deployment hints, and troubleshooting.

# Core components:

Python ingestion pipeline: extract, chunk, embed, index to Qdrant.

Qdrant: vector store with multiple collections/namespaces.

Spring Boot service: orchestration, retrieval logic, RBAC, connectors to Qdrant and Ollama.

Ollama (or other local runtime): embeddings (if needed) & generation.

Streamlit front-end and Teams bot adapter (Teams points to Spring Boot).

# Implementation process — phases & responsibilities

## Design & planning

Inventory data sources (docs, repo list, Confluence/Jira exports, S3 buckets, infra manifests, ELK/log samples).

Define access rules: which teams can see what (sensitivity taxonomy).

Decide initial model(s) you will run locally (choose a 3–7B model for CPU/GPU availability).

## Ingestion pipeline (Python)

Implement connectors/loaders per source type (git repo, markdown, PDF, Confluence, cloud storage).

Preprocessing: text extraction, normalization, language detection, basic cleanup.

Chunking rules (per content type), metadata extraction, sensitivity tagging, secret redaction.

Compute embeddings (consistent model) and upsert vectors into Qdrant collections.

Provide modes: full reindex, incremental reindex (changed files only), webhooks on repo change.

Validate embeddings vs. runtime embedding model compatibility.

## Index design & metadata

Separate collections (recommended): docs_index, code_index, infra_index, logs_index.

Payload metadata fields: source, repo/project, path, chunk_index, last_modified, sensitivity, owner/team.

Query-time filters must use metadata to enforce RBAC and reduce noise.

## Spring Boot orchestration

Provide two retriever flows:

DocsRetriever: queries docs_index only and applies strict filters (no secret chunks).

FullRetriever: queries multiple indices (docs + code + infra + logs) and merges results with score normalization.

Build prompt/assembly logic: template management and token budgeting.

Integrate LLM client adapter: abstract generation API calls to Ollama/TGI/vLLM.

Enforce RBAC + authentication: Teams and Streamlit authentication flows.

Audit and logging service: store userId, retriever used, chunk ids returned (not full text), timestamps.

## UI integrations

Teams Bot:

Register internal bot; restrict to tenant.

Keep it lean: short answers, source links, "read more" behavior.

Streamlit:

Rich chat with context pane showing retrieved chunks (expandable), code snippet viewer, upload for ad-hoc logs, role-aware toggles (include infra/logs).

Allow user to request “actionable steps” and request code patch suggestions.

## Security, governance & privacy

Redact secrets at ingestion and tag sensitivity for each vector.

Teams endpoints only allowed to query docs_index and must never return sensitive content.

Streamlit must require SSO (Azure AD) and be available only inside VPN or internal network.

Implement audit retention, regular privacy review, and data deletion processes.

## Monitoring, testing & operations

Metrics: LLM latency, Qdrant latency, ingestion job success rate, API errors.

Add tracing (OpenTelemetry) across ingestion → retrieval → LLM generation.

Test: unit tests for retriever filters, integration tests with small local model or mocks, security tests for RBAC.

## Rollout

Dev/test pilot with small group and a limited doc set.

Iterate prompts and retrieval tuning.

Expand to more content and users after safety checks.

# Detailed implementation guide (step-by-step tasks, dependencies, acceptance criteria)
## Phase A — Planning & inventory

Tasks:

List all repositories, documentation stores, Confluence spaces, runbooks, infra manifests, logs samples.

Identify owners and sensitivity levels per source.

Decide model(s) to run locally: small/medium (3–7B) for initial pilot.

Acceptance:

Inventory spreadsheet with owners + sensitivity + access rules.

Selected model and runtime documented.

## Phase B — Ingestion pipeline (Python)

Tasks:

Implement source loaders:

Git repos: clone or use GitHub/GitLab API to fetch files and metadata.

Docs: markdown, html, pdf, docx loaders.

Confluence/Jira: export connectors or API-based.

Logs: stream or batch export, pre-summarize noisy logs.

Preprocessing:

Normalize whitespace, remove boilerplate, strip binary content.

Secret/redaction patterns: regexes for keys, tokens, passwords.

Tag sensitivity: public/internal/secret.

Chunking rules:

Docs: ~300–800 tokens with overlap 50–150 tokens (adjust to tokenization).

Code: chunk by function/class or maintain file-based chunks; embed path metadata.

Infra: per resource (manifest) as a chunk.

Logs: aggregate by timeframe or transaction and summarize before embedding.

Embeddings:

Choose embedding model (sentence-transformers or Ollama embedding if available).

Ensure same model for ingestion and query-time embedding to maximize semantic alignment.

Upsert to Qdrant:

Create collections with vector size and distance metric.

Store payload metadata for filtering.

Reindexing strategies:

Full reindex: recreates collections (nightly or as needed).

Incremental: use last_modified and content-hash to upsert changed chunks.

Operationalize:

Wrap ingestion as CLI with config and environment variables.

Provide scheduling (cron, Airflow, or Git webhook triggers).

Acceptance:

A reproducible ingestion run that indexes sample dataset and returns meaningful nearest neighbors for test queries.

## Phase C — Qdrant index & metadata schema

Tasks:

Create collection naming convention and field conventions.

Define required payload keys: source, path, repo, project, sensitivity, last_modified, chunk_index.

Define filtering rules for each collection.

Acceptance:

Qdrant collections populated and sample queries (kNN) return relevant chunks; filters work to exclude secrets.

## Phase D — Spring Boot orchestration & retrievers

Tasks:

Implement retriever abstraction with two concrete retrievers: DocsRetriever and FullRetriever.

DocsRetriever behavior:

Query docs_index only.

Apply filter sensitivity != secret.

k default small (e.g., 5).

FullRetriever behavior:

Query multiple collections, aggregate results, optionally re-rank by contextual match and recency.

k higher (e.g., 10–20); support project scope filtering.

Prompt assembly:

Insert retrieved chunks as context in a template; include chunk ids for traceability.

Enforce token budgeting: sum token estimates for chunks + system + user + model limit.

Decide how to truncate or select chunks when over budget (prefilter on score + recency).

LLM client:

Implement adapter to call Ollama/TGI/vLLM; handle streaming or synchronous return.

Configure timeouts and retries.

Endpoints:

POST /api/chat/teams: accepts userId + message + conversationId; uses DocsRetriever.

POST /api/chat/streamlit: accepts userId + message + mode; uses FullRetriever and returns structured JSON (summary/actions/patch/citations/confidence).

Auditing:

Log which chunk ids were returned and by which user (do not store whole chunk text).

Acceptance:

For sample queries, backend returns relevant answers and lists source chunk ids; RBAC enforcement works (Teams cannot access infra/code).

 ## Phase E — UI integrations

Teams:

Register bot internal tenant, configure messaging endpoint to backend.

Validate requests at backend (Bot JWT / App ID verification) and restrict tenant.

Keep Cards adaptive and concise — return link to doc or "open in Lexora web" button.

Streamlit:

Provide chat view with a context pane showing retrieved chunks and clickable links to source files.

Allow toggles for retrieval scope and an "upload logs" for ad-hoc summarization.

Acceptance:

Teams offers short, accurate answers to doc queries; Streamlit shows structured, actionable answers with sources and code suggestions.

## Phase F — Security, RBAC & governance

Tasks:

SSO integration for Streamlit and admin endpoints (Azure AD/OIDC).

Teams validation via App ID/JWT; restrict tenant.

RBAC mapping: map Azure groups or internal mapping to roles (user/dev/devops/admin).

Enforce Qdrant filters server-side before returning any chunk.

Implement secret redaction and verify no secret payload reaches client.

Acceptance:

Tests demonstrating a user without DEVOPS role cannot retrieve infra_index content; secrets not accessible via Teams.

## Phase G — Monitoring, testing & launch

Tasks:

Instrument major services with metrics (Prometheus) and traces (OpenTelemetry).

Create integration tests: small Qdrant instance + small embedding model + mock LLM or local tiny model.

Run pilot with small group: gather feedback, tune prompts and retrieval parameters.

Acceptance:

Stable service, acceptable latencies for interactive queries, and prompt quality meeting pilot expectations.

# Prompt engineering & templates (practical guidance)

Always include a System message that restricts the model to "use only the context provided" and asks it to cite chunk ids for any factual claims.

Keep Teams prompts concise: instruct model to answer in 3–5 sentences and include a source link.

Streamlit prompts can be structured: ask for (1) 1–2 line summary, (2) ordered action items, (3) suggested code diff, (4) confidence level, (5) list of chunk ids used.

Token budgeting approach: estimate tokens per chunk conservatively; prefer higher-score chunks; prefer recency or doc-authority boost for infra/logs.

When asking for code patches, require the assistant to include file path and exact lines or a minimal patch snippet to reduce ambiguity.

# API contracts & behavioral expectations (no code)

Teams endpoint:

Purpose: short Q&A from docs.

Inputs: authenticated Teams webhook request with user id, tenant id, message, conversation id.

Behavior: use DocsRetriever, assemble concise prompt, call LLM, trim answer to a short length, include list of 1–3 sources (paths + chunk ids), do not include sensitive data.

Expected output: text string answer + array of source references + confidence flag.

Streamlit endpoint:

Purpose: deep technical assistant.

Inputs: authenticated SSO user id, message, optional mode (debug/explain/search), optional project/repo scope.

Behavior: use FullRetriever, return structured JSON containing summary, ordered actions, optional code patch suggestion, list of source chunk ids, confidence.

Expected output: structured response parseable by the UI for rendering code blocks and links to source.

# Security & data governance specifics

Redaction: automatically detect and mask API keys, passwords, private certificates before embedding. If redaction cannot be sure, tag as sensitivity: secret.

Index-level controls: docs_index accessible by Teams; code_index and infra_index only via Streamlit & authorized roles.

Audit: every query logs time, user id, retriever used, chunk ids returned. Implement retention policy.

Network: restrict Qdrant/Ollama services to private network or behind service mesh; backend proxies handle external requests only via validated endpoints.

Review process: owners should be able to mark indices or specific files as "do not index".

# Testing & quality assurance

Unit tests: prompt assembly, token budgeting, metadata-based filter logic.

Integration tests: ingestion → index → retrieval → LLM mock response pipeline.

Security tests: RBAC checks, ensure Teams can't access secret data.

Human eval: sample questions to measure factuality/hallucination rate; iterate prompt & selection heuristics.

Regression testing: after index changes, run standard Q&A tests to detect retrieval drift.

# Deployment & ops: recommended baseline

Use Docker Compose for development and small internal deployments.

Production: containerize services and run behind an internal load balancer or Kubernetes. Keep Qdrant persistent storage on fast block storage.

Scale horizontally: multiple Spring Boot instances, one Qdrant cluster (or sharding/replicas), LLM runtime scaled separately (or use GPU nodes).

Backups: snapshot Qdrant volumes and store audit logs in secure object storage.

# Monitoring & metrics to track

Query throughput and P95 latency for /api/chat/teams and /api/chat/streamlit.

Qdrant query latency and embedding generation time.

LLM generation time and error rates.

Ingestion success rate and time since last index.

Hallucination alerts: e.g., when users flag answers as incorrect; maintain a feedback loop for prompt/retrieval tuning.

# Troubleshooting common issues & remediation

If Qdrant returns low-quality hits: verify embedding model parity between ingestion and query-time; adjust chunking size & overlap; increase top-k or tune re-ranking.

If LLM hallucinates: tighten system prompt to restrict to context; reduce number of chunks; ask the model to explicitly cite chunk ids and penalize answers without sources.

Slow Streamlit responses: implement async job submission with a job-id + polling, or use smaller model for interactive preview then run heavyweight generation asynchronously.

Leakage of secrets: stop ingestion pipeline; audit recent uploads; reindex after redaction rules applied; rotate compromised secrets.

# Rollout & governance plan (recommended sequence)

Internal dev prototype with a small docs corpus and internal users only.

Pilot with a small engineering team; collect feedback and tune prompts/filters.

Expand indexing to code repos and infra for Streamlit pilot (narrow team).

Hardening: RBAC, SSO, audit, secret redaction policy.

Wider rollout on Teams for company-wide docs Q&A.

Ongoing retraining/reindex cadence and governance meetings with data owners.

# Acceptance criteria (for MVP)

Teams bot responds to docs Q&A with accurate answers and cites at least one source chunk.

Streamlit provides structured answers for debugging queries and shows source chunk(s).

RBAC: Teams users cannot access infra/code; unauthorized attempts are logged and blocked.

Ingestion pipeline runs end-to-end and Qdrant contains searchable vectors for the sample corpus.

Basic monitoring and audit logs in place.

# Checklist to start building right now

 Finalize inventory and access list of sources.

 Decide local LLM model & runtime and validate it runs on available infra.

 Implement Python ingestion with chunking & redaction and index small sample set.

 Create Qdrant collections and populate sample vectors.

 Implement Spring Boot skeleton exposing /api/chat/teams and /api/chat/streamlit with retriever stubs.

 Wire LLM adapter and test end-to-end with a small dataset.

 Register Teams bot and point to backend; secure Streamlit behind SSO.

 Run pilot with 2–3 engineers and gather feedback.
