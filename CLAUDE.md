# CLAUDE.md

Guidance for Claude Code when working in this repository. Read this before making changes.

## What this project is

A chatbot that scrapes ongoing/upcoming scholarship listings from the web, stores them in a
queryable data layer, and answers user questions via a RAG-backed agent. Full architecture
and rationale: [`docs/architecture.md`](docs/architecture.md). Diagrams and stack summary:
[`README.md`](README.md).

Everything must stay **open source and free to self-host**, and must run comfortably on a
**local Mac** (not a beefy cloud cluster). When in doubt between a heavier "industry standard"
tool and a lighter equivalent, prefer the lighter one (this is why the project uses k3s over
full kubeadm, Linkerd over Istio, RabbitMQ over Kafka, OpenSearch over Elastic-licensed ELK,
and a local Ollama model over a hosted LLM API).

## Build order — follow the roadmap

Do not jump ahead to Kubernetes/ArgoCD/service-mesh work before the core chatbot logic works
locally. The phase order (see README "Roadmap") is deliberate:

1. Scraper → Postgres + Markdown corpus (plain Python, no infra beyond a local Postgres)
2. FastAPI backend + Ollama + RAG (Qdrant), orchestrated via `docker-compose`
3. CI (GitHub Actions: lint, test, Sonar, Trivy, build, push)
4. k3s/Kind cluster via Ansible, deployed manually with `kubectl` first
5. ArgoCD GitOps migration
6. RabbitMQ + Redis
7. Observability (OTel → Prometheus/Grafana, then OpenSearch/Kibana)
8. Linkerd service mesh + NetworkPolicies
9. Availability CronJob + load testing (k6/Locust)
10. Hardening (secrets management, chaos testing, docs)

If asked to work on a later-phase component before earlier phases are functional, flag this
and confirm before proceeding — it's usually not what's wanted.

## Repository layout

```
scraper/          Scrapy/Playwright spiders, normalizer, embedder workers
backend/           FastAPI app, agent orchestrator (LangGraph/LlamaIndex), tool definitions
shared/            Pydantic schemas shared by scraper + backend — keep both sides in sync
tests/              unit/ and integration/ — mirror the source tree, one test module per source module
data/               git-tracked markdown snapshot of scraped scholarships (human-reviewable corpus)
.github/workflows/ CI pipeline definitions
gitops/             K8s manifests ArgoCD watches (kept in this repo for now; see note below)
ansible/            cluster bootstrap playbooks
docs/               architecture.md — the full design doc
```

**Note on GitOps repo split:** the architecture doc recommends a *separate* gitops repo so CI
and CD stay decoupled. This scaffold keeps `gitops/` in the same repo for a simpler single-repo
start (Phase 1–4). Split it into its own repo once you reach Phase 5 (ArgoCD migration) if you
want the cleaner two-repo pattern — update the CI workflow's "bump image tag" step accordingly
if you do.

## Conventions

- **Python**: format with `ruff format`, lint with `ruff check`. Type hints required on all
  public functions. Pydantic v2 for schemas in `shared/`.
- **Tests**: `pytest`. New backend or scraper logic needs at least one unit test before merge.
  Integration tests that need Postgres/Redis/RabbitMQ/Qdrant should spin them up via
  `docker-compose` (see `tests/integration/`), not mock them away entirely.
- **Commits**: small, one logical change per commit. Reference the roadmap phase in the commit
  body when relevant (e.g. "Phase 2: add Qdrant retrieval tool").
- **Secrets**: never commit real secrets. Use `.env` (gitignored) locally; use Sealed Secrets
  or SOPS+age once secrets move into `gitops/`.
- **K8s manifests**: Kustomize preferred over raw YAML duplication; one overlay per environment
  if/when more than "local" exists.
- **Commit messages for gitops changes**: keep image-tag bump commits separate from manifest
  logic changes, so ArgoCD sync history stays readable.

## Things to always check before finishing a task

- Does this change keep the whole stack runnable within reasonable RAM on a single Mac? Heavy
  additions (another stateful service, a bigger model) should be called out, not silently added.
- Does new backend logic emit OpenTelemetry spans consistent with what already exists (once
  Phase 7 is underway)?
- Does a new scraper source respect robots.txt and reasonable rate limits?
- If touching `shared/schemas`, are both `scraper/` and `backend/` updated to match?

## What not to do

- Don't introduce a paid API or SaaS dependency (LLM APIs, hosted vector DBs, etc.) — everything
  must be self-hostable and free.
- Don't add Istio, Kafka, or other heavy alternatives to the lightweight tools already chosen,
  without discussing the tradeoff first — resource budget on a laptop is the binding constraint.
- Don't skip the Trivy/Sonar CI steps to "make the pipeline pass" — fix the underlying issue or
  explicitly document why a finding is accepted risk.
