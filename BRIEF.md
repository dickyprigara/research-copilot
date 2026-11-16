# PROJECT BRIEF

This file is the specification for the project. It lives at `docs/BRIEF.md`. Claude Code: read all of it before doing anything, then follow section 12.

Everything in this brief is a decided default unless it is listed under "Questions to ask me first" (section 12). If you think a default is wrong, say so and record the change in `docs/adr/`.

---

## 1. Goal

Build one portfolio product that shows I can take an AI product from idea to a running, observable, cost-tracked deployment:

> **A research workspace that turns real customer conversations into market research and competitor analysis, using RAG with citations and an agent that can query external data, installed on Kubernetes with Grafana dashboards that show what every answer costs.**

Working name: `MarketLens` (suggest better ones in your first reply; the name must be a single config value, not hardcoded).

## 2. About me

- I am a product/project manager with limited coding experience. You write the code. I must be able to explain every design decision in an interview.
- Explain important files and choices in plain language. No unexplained config: if you set a memory limit or a chunk size, say why.
- Never invent results. Every number I will show (cost per query, retrieval accuracy, latency, savings) must come from a measurement that is reproducible from the repo.

## 3. Hard constraints (design around these)

1. **I have no local development machine.** No local Docker, no local cluster. I work through Claude Code on the web and the GitHub website.
2. **Your sandbox may not have Docker or a cluster.** Do not assume it does. Check, and plan verification accordingly.
3. **Cluster-level verification runs in GitHub Actions** using a throwaway `kind` cluster: build images, `helm install`, run a smoke test. A green check on the pull request is how I know a phase works.
4. **The real deployment target is one small Linux VPS** that I reach by SSH, running k3s. The VPS already runs other services of mine, so the installer must check free RAM, free disk and port conflicts (80, 443, 6443) first, and stop with a clear message instead of breaking anything.
5. **I have little time.** Build the Core scope (section 10) first. Stretch items wait until Core is done.
6. **Money.** The Anthropic API is billed per use. Keep a hard daily budget in config (default 1 USD per day) and make all automated tests run with a fake LLM client, so CI needs no API key and costs nothing.
7. **Sessions end.** Keep `docs/PLAN.md` with checkboxes current after every phase, so a new session can continue from it.

## 4. The product

**Target user:** a founder or product manager who wants to validate a product idea.
**Demo domain:** project management software for small teams (change only if I tell you).

Flow:

1. **Create a research project**: research question, target audience, optional list of competitors.
2. **Add sources.** Two kinds:
   - *Voice of customer*: forum threads, social posts, video comments, community chats, product reviews. Added by pasting text, uploading files (TXT, MD, CSV, PDF), or pasting a URL of a public page.
   - *Reference*: an open-licensed textbook that supplies the frameworks (for example OpenStax "Principles of Marketing"; confirm the licence on its page before using it).
3. **Run the market research pipeline** (section 5) and get a cited report.
4. **Run the competitor analysis pipeline** (section 5) and get a cited report with a competitor matrix.
5. **Ask**: follow-up questions answered only from the sources, with citations. If the sources do not contain the answer, say so.
6. **Agent mode**: questions that need more than the documents, such as combining findings with structured data or a public API.
7. **Export** reports as Markdown.

Source collection rules:
- Paste and upload come first. They work for every platform and respect each site's terms.
- No scraping of sites that forbid it. Optional connectors may only use official APIs and are Stretch.
- The repo ships a small **synthetic** sample dataset, clearly labelled synthetic, used only by tests and CI. Real collected data stays out of the public repo (`data/private/`, git-ignored). Portfolio findings must come from real data.

## 5. Research method (Tutorial A)

Source: "Market research is easy, actually", Steph France, https://youtu.be/magQlXqwy8E

The stages below are derived from the video's published description and chapter list. The exact prompts in the video were not available. If I add my own notes in `docs/tutorial-notes/market-research.md`, they override this section.

The method is voice-of-customer research: collect what real customers say, organise it in a grounded AI notebook, then extract problems, language, solutions and personas. In this product, the RAG knowledge base plays the role the notebook tool plays in the video.

| # | Stage | Market research output | Competitor analysis output (same logic, competitor as the unit) |
|---|---|---|---|
| 1 | Define | Audience and research question | Competitor set and comparison question |
| 2 | Collect | Conversations about the problem | Conversations and reviews about each competitor, plus their public pricing and feature pages |
| 3 | Organise | Ingested, chunked, tagged by source and platform | Same, also tagged by competitor |
| 4 | Problems | Pain points, clustered, with how often each appears and verbatim quotes | What customers praise and complain about, per competitor |
| 5 | Language | Glossary of the words customers use | Words used when comparing products; reasons for switching |
| 6 | Solutions | Solution ideas tied to each pain point | Unmet needs and gaps no competitor covers |
| 7 | People | Personas and ideal customer profiles | Which segments each competitor serves well or badly |
| 8 | Messaging | Value propositions and sample marketing copy | Positioning and differentiation statements |
| 9 | Report | Cited market research report | Cited report with competitor matrix and recommendations |

Implementation rules:
- One pipeline engine runs both pipelines. Each stage is a file in `config/pipelines/` with: inputs, prompt template, allowed tools, output schema (validated JSON), and citation rule. I must be able to change the method by editing these files.
- Stages 4 and 5 are evidence: every item needs at least one citation to a chunk, and quotes must be verbatim from the source.
- Stages 6 to 8 are ideas generated by the model. Label them as generated in the UI and the report, and link each back to the evidence it rests on.
- Counts such as "appears in 14 of 120 comments" must be computed in code from retrieved data, not estimated by the model.

## 6. Design (Tutorial B)

Source: "New Claude Code for Web Designers: Full Course", Fadel, https://youtu.be/zn0mn8thIGk

Derived from the video's published description and chapters. If I add `docs/tutorial-notes/design.md` or the course's own `design-spec.md`, that overrides this section. Follow the tutorial's **process**, not the look of its example site.

1. Write a one-page client brief for this app (who uses it, what they must do, what "good" looks like).
2. Describe wireframes in text for each screen, phone layout first.
3. Write `docs/design/design-spec.md`: colour tokens for light and dark themes, text contrast checked against WCAG AA, typography (Inter), spacing scale, grid, and components (header, footer, buttons, forms, cards, tables, citation chip).
4. Show me the spec and wait for approval before building screens.
5. Build from the spec, with tokens in one file so the look is changeable in one place. Include a light/dark switch.
6. Finish with an accessibility check and a Lighthouse run in CI. Record the scores; do not promise specific ones.

Screens: project list, project setup, sources, pipeline run with progress, report view with clickable citations, ask/agent chat, and a cost panel showing what this project has spent.

## 7. Architecture (decided)

| Part | Choice | Reason |
|---|---|---|
| Backend | Python, FastAPI. One image, run as `api` or `worker` by command | Fewer images to build and explain |
| Frontend | React, Vite, TypeScript, Tailwind, built to static files and served by nginx | Light enough for a small VPS |
| Data | PostgreSQL with pgvector, one instance | App data, vectors, job queue, cost ledger and eval results in one place |
| Jobs | Postgres table used as a queue (`SELECT ... FOR UPDATE SKIP LOCKED`) | No extra service |
| LLM | Anthropic API. Model IDs and prices in config, never in code | Read the current documentation at docs.claude.com for model names, tool use and prompt caching before coding |
| Embeddings | Small local CPU model (for example via `fastembed`), behind an interface | No second API key, no extra cost |
| Retrieval | Vector search plus Postgres full-text search, merged | Better recall than vectors alone |

### RAG rules
- Every chunk keeps source, platform, page or position, section and competitor tag.
- Heading-aware chunking for documents; one comment or review per chunk for conversations. Size and overlap are config values.
- Re-ingesting the same file must not create duplicates.
- Answers use only retrieved context and cite it. "Not found in the sources" is a valid answer.

### Agent rules
- Tool-use loop with limits on tool calls, tokens and time per request.
- Tools, each with a strict schema and tests:
  1. `search_knowledge_base`
  2. `query_insights_db`: read-only SQL role, allow-listed views (pain points, competitor matrix), row limit
  3. `fetch_public_indicator`: one allow-listed public API (default World Bank indicators, for market context), response size cap
- Tool results are data, never instructions. Include a test where a tool result contains an instruction and the agent must not follow it.
- No write tools. Every tool call is logged with duration and cost.

### Evaluation
- `eval/` holds 30 or more questions with expected answer notes and source references, including questions whose answer is not in the sources.
- Measure retrieval hit rate, citation correctness, answer correctness, correct refusals, latency and cost per question.
- One command produces a Markdown report. Draft the question set, then tell me which items I must check by hand.

## 8. Observability and cost

- **Metrics** at `/metrics` (Prometheus): requests, latency, errors, queue depth, ingestion time, retrieval time, tool calls by tool and result, agent steps per request, and LLM tokens (input, output, cached) and cost in USD, labelled by project, pipeline and model.
- **Cost ledger**: one row per LLM call in Postgres. Cost is tokens multiplied by prices from `config/pricing.yaml`. Add a comment in that file telling me to confirm current prices.
- **Budget**: when the daily budget is reached, the API refuses new LLM work with a clear message.
- **Dashboards**, shipped inside the Helm chart so they appear automatically:
  1. App health
  2. LLM usage and cost, including cost per query and the most expensive requests
  3. Infrastructure cost by workload from OpenCost
  4. **Total cost per query** = LLM cost + infrastructure share. This is the headline number.
- **Alerts**: error rate, slow responses, agent looping, daily spend near budget, pod restarts, disk nearly full.
- **Logs**: structured JSON with a request ID.

## 9. Kubernetes and delivery

- Dockerfiles: multi-stage, slim base, non-root user, pinned dependencies.
- One Helm chart at `deploy/helm/marketlens`: api, worker, web, Postgres with a persistent volume, Ingress, ConfigMap, Secret reference, CronJob placeholder for data refresh, resource requests and limits and health probes on every container, standard labels so OpenCost can attribute cost.
- Monitoring behind a switch: `monitoring.enabled` adds ServiceMonitor, alert rules and dashboards.
- Values files: `values.yaml`, `values-ci.yaml` (kind in GitHub Actions), `values-vps.yaml` (small k3s node).
- **Sizing**: propose resource numbers that fit the VPS after I tell you its RAM. Offer a "lite" monitoring profile (short retention, no extra replicas) and tell me plainly if the VPS is too small.
- **VPS install**: `scripts/install-vps.sh` runs the safety checks from section 3, installs k3s and Helm if missing, then the monitoring stack, OpenCost and the app. One command over SSH. Also `scripts/uninstall-vps.sh`.
- **Public safety**: the app and Grafana must not be open to the internet without a login. Put basic auth on the Ingress and rate-limit the API, so strangers cannot spend my API budget.
- **Secrets**: never in the repo. The API key goes into a Kubernetes Secret created by the install script from a prompt.
- **CI** (GitHub Actions): lint, type check, tests, build images, vulnerability scan, `helm lint`, install into `kind`, smoke test, push images to GHCR on merge.

## 10. Scope

**Core (build this first):** everything in sections 4 to 9.

**Stretch (only after Core is finished, in this order):**
1. Expose the agent tools as an MCP server
2. Langfuse traces for prompts and retrieved chunks
3. One measured efficiency comparison beyond the Core one (model routing or prompt caching)
4. Autoscaling, NetworkPolicies, PodDisruptionBudget
5. Official-API connectors for collecting sources
6. PDF export, Loki, OpenTelemetry tracing, load test, Argo CD

## 11. Phases

One phase = one pull request. Each PR description has: what was built, why, and a "How to check" list I can follow from a browser.

| # | Phase | How I check |
|---|---|---|
| 0 | Skeleton: repo layout, `CLAUDE.md`, `docs/PLAN.md`, CI with kind smoke test, hello API in the Helm chart | Green check on the PR |
| 1 | Ingest sources and ask with citations, with a bare UI | CI integration test on the synthetic data |
| 2 | First VPS install with the bare app | I run one command over SSH and open the page |
| 3 | Evaluation harness and baseline numbers | Eval report committed |
| 4 | Market research pipeline | Report generated in CI with the fake LLM; real run on the VPS |
| 5 | Competitor analysis pipeline | Same |
| 6 | Design spec, then the real UI | I approve the spec, then view the deployed UI |
| 7 | Agent and tools, with safety tests | Injection test passes; a question that needs two tools works |
| 8 | Metrics, dashboards, cost ledger, OpenCost, alerts | Dashboards show real traffic on the VPS |
| 9 | One efficiency experiment with before and after numbers, README, architecture diagram, runbook | A stranger could install it from the README |

## 12. How to work

1. **First reply: no code.** Give me, in this order:
   a. Whether your sandbox has Docker, and how that affects verification.
   b. The questions below, plus anything else that blocks you.
   c. Risks, and defaults you disagree with.
   d. A proposed `docs/PLAN.md`.
2. After I approve, do Phase 0 only. Then one phase at a time, stopping after each.
3. Put the working rules into `CLAUDE.md` in Phase 0 if the file is missing.
4. Record significant decisions in `docs/adr/` with the alternatives you considered.
5. If you cannot confirm a library version, an API detail or a price, check the documentation or tell me. Do not guess.
6. If a step needs me to do something outside the repo (create an API key, run a command on the VPS, change a GitHub setting), give me exact click-by-click or copy-paste steps.

**Questions to ask me first:**
- VPS: operating system, RAM, CPU, free disk, and what already runs on it
- Whether I have a domain name, or the app is reached by IP address
- Whether I already have an Anthropic API key with credit
- Whether the repo should be public (free CI minutes, portfolio visibility) or private
- Whether I have notes or the asset pack from either tutorial

## 13. Done means

- The chart installs in CI on every pull request and on the VPS with one command.
- Both pipelines produce reports where every evidence item has a working citation.
- The agent answers a question that needs two different tools, inside its limits.
- Grafana shows app health, LLM cost, infrastructure cost and total cost per query from real use.
- The eval report and one before/after efficiency result are committed, with real measurements.
- The README explains the problem, the architecture, how to install, the results, the limits and what comes next.
