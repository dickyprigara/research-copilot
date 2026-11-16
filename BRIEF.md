# PROJECT BRIEF FOR CLAUDE CODE

> Save this file in the repository as `docs/BRIEF.md`, then tell Claude Code to read it.
> Fill in the `{{ ... }}` slots when you can. Slots A and B are the two tutorials.

---

# TIME NOTE (read first)

I have little time right now. Slots A and B are not filled yet.
- Plan and build Phases 0, 1, 2 first (skeleton, RAG with citations, eval harness). They don't depend on the tutorials.
- For Phases 3 and 5, use a sensible generic default (a standard market research flow, a clean minimal UI) and keep both in `config/` and a design-token file, so I can replace them later when I provide the tutorial notes.
- Ask me for Slot A and Slot B content only when we reach Phase 3 and Phase 5.

---

# ROLE

You are my senior engineer and technical mentor. We are building one portfolio-grade product together, from an empty repo to a Kubernetes deployment with observability and cost tracking.

# ABOUT ME (adjust how you work accordingly)

- I am a product/project manager with limited coding experience. I rely on you to write code, but I must be able to explain every design decision in a job interview.
- Therefore: explain each important file and decision in plain language, record trade-offs in `docs/adr/`, and never leave "magic" config unexplained (for example, why a given memory limit or chunk size was chosen).
- Do not invent results. Any number I show in the portfolio (cost per query, retrieval accuracy, latency, savings) must come from something we actually measured and can reproduce.

# PRODUCT

Working name: `{{PROJECT_NAME}}` (suggest 3 alternatives if I leave this blank).

**One-line description:** an AI research workspace that runs **market research** and **competitor analysis** over a private knowledge base using **RAG**, with an **agent** that can also query external data sources, deployable on **Kubernetes** with full **observability and cost transparency**.

**Target user:** `{{e.g. a product manager or founder validating a product idea in Indonesia / SaaS / fintech}}`
**Initial domain for demos:** `{{DOMAIN}}`

## What the product does

1. **Projects.** A user creates a research project with a research question and a domain.
2. **Knowledge base (RAG).** The user adds sources: uploaded PDFs (including open-licensed textbooks), pasted URLs, and web search results. The system extracts text, chunks it with page and section metadata, embeds it, and stores it.
3. **Market research pipeline.** Follows the logic of Tutorial A (Slot A below). Produces a structured, cited market research report.
4. **Competitor analysis pipeline.** Reuses the same pipeline engine and logic as market research, with competitors as the unit of analysis (Slot A logic, adapted). Produces a competitor matrix, customer-voice themes, SWOT, positioning, and a gap analysis with recommendations.
5. **Chat / ask mode.** The user asks follow-up questions answered only from the knowledge base, with citations (source, page/section). The system must say "not found in the sources" instead of guessing.
6. **Agent mode.** An agent decides which tool to use: search the knowledge base, query an external database or API, or read the competitor dataset. It reasons over the results and answers with citations.
7. **Exports.** Reports export to Markdown and PDF (DOCX optional).

---

# SLOT A: MARKET RESEARCH LOGIC (tutorial 1)

Source: https://youtu.be/magQlXqwy8E

`{{PASTE HERE: the steps, framework, prompts and outputs the tutorial uses. A transcript or my own bullet summary is fine.}}`

Instructions for you (Claude Code):
- Treat the pasted material as the specification of the research pipeline. Convert it into an explicit ordered list of pipeline stages, each with: input, the prompt template, the tools it may call, the output schema, and the citation requirement.
- Show me that list and wait for my confirmation before implementing the pipeline.
- Do not fill gaps from memory of what such tutorials usually say. If something is unclear, ask me.
- For competitor analysis, show me explicitly how each stage maps (for example, "market sizing" becomes "competitor positioning and pricing"), and get my approval of the mapping.

# SLOT B: DESIGN (tutorial 2)

Source: https://youtu.be/zn0mn8thIGk

`{{PASTE HERE: the design approach, style, layout, tools, color/typography rules, and screens from the tutorial. Screenshots are welcome.}}`

Instructions for you (Claude Code):
- Extract a short design brief from it: visual style, color tokens, typography, spacing, component list, and the key screens.
- Implement it as a design-token file and a reusable component set so the style is changeable in one place.
- Show me the brief before building the full UI.

---

# ARCHITECTURE (defaults; challenge them if you have a better reason, and record it in an ADR)

| Layer | Default choice | Why |
|---|---|---|
| Backend API | Python 3.12, FastAPI | Best ecosystem for LLM and RAG work |
| Frontend | Next.js (TypeScript) or React + Vite, following Slot B | Matches the tutorial design |
| Database | PostgreSQL with pgvector | One store for app data, vectors, the cost ledger, and eval results; fewer moving parts on Kubernetes |
| Background jobs | A worker process using a Postgres- or Redis-backed queue; heavy one-off ingestion as Kubernetes Jobs | Ingestion and report generation are slow |
| LLM | Anthropic API (Claude) | Tool use for the agent. Model names come from env/config, never hardcoded |
| Embeddings | `{{CHOICE: hosted embedding API | local sentence-transformers}}` | Configurable behind an interface |
| Agent tools | Exposed as an **MCP server** and also callable in-process | Standard integration layer, strong portfolio signal |
| Search | Hybrid: vector similarity + Postgres full-text, optional reranker | Better retrieval than vector alone |

Check the current Anthropic documentation (docs.claude.com) for the current tool-use, prompt caching, and MCP guidance before implementing, rather than relying on memory.

## RAG requirements

- Ingestion keeps `source_id`, page number, section/heading, and chunk index for every chunk.
- Chunking is heading-aware, with configurable size and overlap (start around 500 to 800 tokens, 10 to 15% overlap) and these as parameters so we can experiment.
- Every generated claim carries a citation back to a chunk. Reports render citations as clickable references.
- Answer-only-from-context prompt with explicit "not found" behavior.
- Idempotent re-ingestion (re-uploading the same file does not duplicate chunks).

## Agent requirements

- Tool-use loop with hard limits: max tool calls per request, max tokens per request, per-request timeout.
- Tools (each with a strict JSON schema and its own tests):
  1. `search_knowledge_base(query, filters)`: RAG retrieval
  2. `query_external_database(...)`: **read-only**, allow-listed tables/views only, parameterized queries, row limit. Default demo target: a second Postgres/SQLite with competitor or market data
  3. `fetch_external_api(...)`: allow-listed domains only, with a response size cap. Demo target: `{{e.g. World Bank API, CoinGecko, a news API}}`
  4. `read_competitor_dataset(...)`: the structured output of the competitor pipeline
- Prompt-injection defense: tool results are treated as data, never as instructions. Add a test that feeds a malicious string through a tool result and asserts the agent does not obey it.
- Any write or outbound action requires explicit human confirmation. Default: the agent has no write tools.
- Every tool call is logged (inputs, outputs truncated, duration, cost).

## Evaluation harness (required)

- `eval/` folder with a question set (target 30 to 50 questions, each with expected answer notes and the source page/section).
- Metrics: retrieval hit rate@k, citation correctness, answer correctness (LLM-judge plus manual spot-check), "correctly abstains" rate, latency, cost per question.
- A CLI command runs the evaluation and writes a Markdown report. Compare at least two configurations (for example chunk size, or with and without reranking) and keep the results in the repo.
- Draft the question set from the ingested source, then tell me exactly which items I must review by hand.

---

# OBSERVABILITY AND COST TRANSPARENCY

## Metrics (Prometheus, exposed at `/metrics`)

At minimum: request count, latency histograms, error counts, queue depth, ingestion duration, retrieval latency, tool-call counts by tool and status, agent steps per request, and these LLM metrics labeled by `project`, `model`, `pipeline` (market_research / competitor / chat / agent):
- `llm_input_tokens_total`, `llm_output_tokens_total`, `llm_cached_tokens_total`
- `llm_cost_usd_total`, computed as tokens times a **price table kept in config** (`config/pricing.yaml`, with a note that I must verify current prices). Never hardcode prices in code.
- `cost_per_query_usd` (histogram)

## Tracing and LLM observability

- OpenTelemetry instrumentation across API, worker, retrieval, and tool calls.
- Langfuse integration (make it optional via env flag; document both self-hosted and hosted-free-tier setup). Capture prompts, retrieved chunks, tool calls, tokens, cost, and latency per trace.

## Logs

Structured JSON logs with a request/trace ID. Optional Loki via the Helm chart toggle.

## Dashboards (Grafana, shipped as JSON in ConfigMaps so they load automatically)

1. **App health:** latency p50/p95, errors, queue depth, pod restarts.
2. **LLM usage and cost:** tokens, cost per day, cost per project, **cost per query**, most expensive questions, cache hit share.
3. **Infrastructure cost:** OpenCost data by namespace and by `project` label, idle versus used resources.
4. **Combined:** infra cost plus LLM cost gives **total cost per query**. This is the headline panel.

## Alerts (PrometheusRule)

High error rate, p95 latency breach, agent tool-call loop (steps per request above threshold), daily LLM spend above budget, pod crash loop, vector DB disk almost full.

## Cost controls in the product

- Per-project daily budget in config; the API refuses or degrades gracefully when exceeded.
- Prompt caching for repeated system prompts and tool definitions.
- Model routing: a smaller/cheaper model for simple steps, a stronger one for synthesis (configurable).
- Embedding cache keyed by content hash.
- Bulk ingestion can use the batch API where supported.

---

# KUBERNETES AND DELIVERY

## Requirement: easy to install

A new user with a cluster must be able to install with **one Helm command** plus a values file, and a local demo with **one `make` command**.

## Deliverables

- Multi-stage Dockerfiles for api, worker, frontend: slim base images, non-root user, pinned dependencies, healthcheck.
- Helm chart `deploy/helm/{{PROJECT_NAME}}`:
  - Deployments (api, worker, frontend), Service, Ingress, ConfigMap, Secret references (support existing secrets and external-secrets), PVC for Postgres/pgvector (or an optional bundled subchart with a toggle for an external DB)
  - **Resource requests and limits on every container**, plus liveness, readiness, and startup probes
  - HPA (api and worker), PodDisruptionBudget, NetworkPolicies, ServiceAccounts with least privilege
  - Standard labels on everything: `app.kubernetes.io/*` and `project={{PROJECT_NAME}}` so OpenCost can attribute cost
  - `ServiceMonitor`, `PrometheusRule`, and Grafana dashboard ConfigMaps behind a `monitoring.enabled` toggle
  - CronJob for scheduled data refresh (competitor and market datasets)
  - `values.yaml`, `values-local.yaml` (laptop sizing), `values-demo.yaml` (small VPS sizing, `{{k3s on VPS / managed cluster}}`)
- `deploy/monitoring/`: install instructions and values for `kube-prometheus-stack`, OpenCost, optional Loki, optional Langfuse (documented as a one-liner each).
- `Makefile` targets: `make dev-up` (create a local k3d/kind cluster, build images, install everything), `make dev-down`, `make test`, `make eval`, `make load-test`, `make deploy-demo`.
- GitHub Actions: lint, test, build images, scan with Trivy, push to GHCR, helm lint/template check. Argo CD manifests are **optional** and come last.
- `docs/RUNBOOK.md`: install, upgrade, debug, what each alert means, how to read the cost dashboards.

---

# REPOSITORY STRUCTURE (propose changes if you have good reason)

```
/
├── apps/
│   ├── api/            # FastAPI: routes, services, pipelines, agent, metrics
│   ├── worker/         # ingestion + report generation
│   ├── web/            # frontend (Slot B design)
│   └── mcp-server/     # agent tools exposed over MCP
├── packages/           # shared libs (config, schemas, llm client, cost accounting)
├── eval/               # question sets, runners, reports
├── deploy/
│   ├── helm/
│   ├── monitoring/
│   └── k3d/            # local cluster config
├── config/             # pricing.yaml, pipeline definitions, prompt templates
├── docs/               # README assets, ARCHITECTURE.md, RUNBOOK.md, adr/
├── scripts/
├── Makefile
└── README.md
```

Pipeline stages and prompts live in `config/` as versioned files (not buried in code), so I can read and edit them.

---

# SECURITY AND QUALITY RULES

- No secrets in the repo, ever. `.env.example` only. Secrets via Kubernetes Secrets or external-secrets.
- Read-only DB credentials and table allow-list for the agent. Domain allow-list for fetch tools.
- Input limits (file size, page count), safe PDF handling, and rate limiting on the API.
- Dependency pinning, Trivy scan in CI, non-root containers.
- Tests: unit tests for chunking, citations, cost accounting, and tool schemas; integration tests for the RAG path; the prompt-injection test above; a smoke test that runs inside the local cluster.
- Typed code, linting (ruff, mypy or pyright, eslint), pre-commit.

---

# HOW TO WORK (important)

1. **Do not write code yet.** First read this whole prompt, then reply with:
   a. Your clarifying questions (everything ambiguous or missing, especially in Slots A and B).
   b. Any risks or places where you disagree with my defaults.
   c. A proposed `PLAN.md` with phases, each ending in something demoable.
2. Wait for my approval of the plan.
3. Work in **vertical slices**, one phase at a time. After each phase: run the tests, show me how to verify it myself, summarize in plain language what you built and why, then pause for my go-ahead.
4. Keep a `DECISIONS.md` or `docs/adr/` entry for every significant choice with the alternatives considered.
5. If something I asked for is a bad idea, say so and propose an alternative instead of silently complying.
6. If you cannot verify a library version, API, or pricing, say so and check the docs rather than guessing.

## Phases (propose adjustments in PLAN.md)

| Phase | Outcome | How I verify |
|---|---|---|
| 0 | Repo skeleton, tooling, CI, local k3d cluster, hello-world API deployed via Helm | `make dev-up`, open the health endpoint |
| 1 | Ingestion plus RAG Q&A with citations on one document | Ask 5 questions, check page citations |
| 2 | Evaluation harness plus first baseline numbers | `make eval` produces a report |
| 3 | Market research pipeline (Slot A) producing a cited report | Generate a report, inspect every citation |
| 4 | Competitor analysis pipeline reusing the engine | Competitor matrix plus gap analysis |
| 5 | Frontend following Slot B, end to end | Run the full flow in the browser |
| 6 | Agent plus MCP tools plus external DB/API plus safety tests | Demo question that needs two tools; injection test passes |
| 7 | Metrics, Langfuse, Grafana dashboards, alerts | Dashboards populated by real traffic |
| 8 | OpenCost plus LLM cost metrics plus combined cost-per-query panel | One number I can explain |
| 9 | Efficiency experiments (right-sizing, prompt caching, model routing, chunk/context size), load test with k6, before/after report | Table of measured deltas |
| 10 | Docs, architecture diagram, runbook, demo script, README polish | A stranger can install it from the README |

# DEFINITION OF DONE

- One-command local install works from a clean machine; the Helm install works on a fresh cluster.
- Both pipelines produce cited outputs from the knowledge base.
- Agent uses at least two different external tools in one answer, within its safety limits.
- Grafana shows app health, LLM cost, infra cost, and total cost per query from real traffic.
- Evaluation report and before/after optimization report are committed with real measurements.
- README covers: problem, architecture diagram, how to run, results, limitations, and what I would build next.

# FIRST MESSAGE TO ME

Start by confirming you have read this, then give me your clarifying questions, risks, and the proposed `PLAN.md`. Do not start implementing.
