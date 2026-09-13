# Scholarship Chatbot — Full Architecture

This is the detailed design reference for the project. For the quick summary and diagrams, see
the root [README.md](../README.md). For instructions on how to work in this repo, see
[CLAUDE.md](../CLAUDE.md).

## 0. Design constraints

- Deploy target: a local Mac (not cloud). Every tool choice is made to keep resource usage
  realistic on a laptop.
- 100% open source / free to self-host — no paid SaaS required anywhere in the stack.
- Built in phases across a year, not all at once (see README roadmap).
- LLM runs locally via Ollama — zero API cost, zero external dependency, but leave the
  interface pluggable in case you want to swap to a hosted API later.

## 1. Components

| Layer | Component | Tool | Why |
|---|---|---|---|
| Scraping | Crawler | Scrapy (static) + Playwright (JS-heavy) | Free, mature, covers most scholarship sites |
| Scraping | Scheduling | Kubernetes CronJob | Native, no extra infra |
| Data | Structured store | PostgreSQL | Free, reliable; can also host pgvector to avoid a separate vector DB |
| Data | Vector store | Qdrant | Open source, lightweight, RAG-native |
| Data | Corpus snapshot | Markdown files in git | Human-reviewable, versioned, cheap fallback context |
| Data | Cache | Redis | Caches LLM responses and frequent queries |
| Queue | Messaging | RabbitMQ | Lighter than Kafka for a single-node Mac |
| Backend | API | FastAPI | Async, lightweight, easy to instrument |
| Backend | Agent framework | LangGraph / LlamaIndex Agents | Open source, tool-calling, RAG-native |
| Backend | LLM | Ollama (Llama 3.1 8B / Mistral 7B, quantized) | Fully local, zero cost |
| Backend | Embeddings | nomic-embed-text / bge-small via Ollama/HF | Free, local, no rate limits |
| CI | Build/test | GitHub Actions | Free minutes for small/public repos |
| CI | Code quality | SonarQube Community (self-hosted) or SonarCloud | Static analysis, quality gate |
| CI | Image scanning | Trivy | Scans built image, outputs SARIF/JSON, fails on critical CVEs |
| CD | GitOps | ArgoCD | Declarative, pull-based deploys from Git |
| CD | Cluster bootstrap | Ansible | Provisions nodes, installs base tooling, bootstraps ArgoCD/secrets |
| Orchestration | Kubernetes | k3s or Kind | Real K8s API, far lighter than full kubeadm |
| Networking | Service mesh | Linkerd | mTLS + traffic policy, low resource footprint |
| Observability | Logs | OpenSearch + OpenSearch Dashboards | Apache-2.0 ELK-compatible fork |
| Observability | Traces/metrics pipeline | OpenTelemetry Collector | Vendor-neutral instrumentation |
| Observability | Trace backend | Jaeger | Distributed tracing UI |
| Observability | Metrics | Prometheus + Grafana | Dashboards, alerting |
| Testing | Availability | K8s CronJob hitting /healthz + sample query | Cheap synthetic monitoring |
| Testing | Load | k6 (Grafana OSS) or Locust | Free, scriptable |
| Secrets | Secret mgmt | Sealed Secrets or SOPS+age | Keep K8s secrets encrypted in git |
| Registry | Container registry | GitHub Container Registry (GHCR) | Free image hosting |

Note on ELK licensing: Elasticsearch/Kibana 7.11+ are under the non-OSS Elastic License. Use
OpenSearch/OpenSearch Dashboards (Apache 2.0) instead — a drop-in ELK-compatible replacement.

## 2. Data flow

1. A CronJob triggers scraper pods on a schedule (every 6-12h).
2. The scraper pushes raw pages/listings into RabbitMQ (`scrape.results`).
3. A normalizer worker consumes the queue, deduplicates against Postgres (URL hash + fuzzy
   title match), writes structured rows to Postgres, and writes a clean Markdown snapshot per
   scholarship into `data/`.
4. New/changed records are pushed onto `embed.jobs`; an embedding worker chunks the markdown,
   generates embeddings locally, and upserts into Qdrant.
5. A user message hits FastAPI. The agent orchestrator decides whether it needs a structured
   DB lookup, a vector search, both, or neither.
6. Retrieved context + the question go to the local Ollama LLM for the final answer.
7. Common query responses are cached in Redis (TTL ~1h, since scholarship data changes).
8. Every request is traced via OpenTelemetry; logs ship to OpenSearch; metrics are scraped by
   Prometheus.

## 3. CI pipeline

```
on: push / pull_request
  1. Checkout
  2. Lint (ruff) + unit tests (pytest) + coverage
  3. SonarQube/SonarCloud scan -> quality gate
  4. Build Docker image (minimal base: python:slim or distroless)
  5. Trivy scan the built image
       - fail on CRITICAL/HIGH CVEs (configurable threshold)
       - export report as trivy-report.json / SARIF -> upload as build artifact
       - optionally upload SARIF to GitHub Security tab
  6. Push image to GHCR (tag: git-sha + semver)
  7. Bump image tag in gitops/ (this commit is what ArgoCD watches)
```

## 4. CD pipeline

- Ansible bootstraps k3s (via Colima/Multipass on Mac), installs ArgoCD, Linkerd, cert-manager,
  Sealed Secrets — idempotent, so the whole cluster can be torn down and rebuilt quickly.
- ArgoCD watches `gitops/` using the app-of-apps pattern: one root Application declares child
  Applications for every service (scraper, backend, redis, rabbitmq, postgres, qdrant,
  observability stack, linkerd).

## 5. Kubernetes namespace layout

| Namespace | Contents |
|---|---|
| `scholarship-data` | Postgres, Qdrant, Redis, RabbitMQ |
| `scholarship-scraper` | Scraper Deployment/CronJob, normalizer, embedder workers |
| `scholarship-backend` | FastAPI app, agent service |
| `scholarship-obs` | OpenSearch, Dashboards, OTel Collector, Jaeger, Prometheus, Grafana |
| `scholarship-ci-tools` | Self-hosted SonarQube / Harbor, if not using SaaS-free-tier equivalents |
| `linkerd` | Service mesh control plane |
| `argocd` | ArgoCD control plane |
| `scholarship-testing` | Load test jobs (k6), availability-check CronJob |

Use Linkerd mesh injection per namespace for automatic mTLS + traffic metrics, and K8s
NetworkPolicies to explicitly restrict cross-namespace traffic (e.g. only
`scholarship-backend` may reach Postgres/Redis/Qdrant).

## 6. Availability & load testing

- Availability CronJob (every 1-5 min): hits `/healthz`, `/ready`, a sample chat query
  end-to-end, and checks Postgres/Redis/RabbitMQ connectivity. On failure, posts to a webhook
  (ntfy, Slack, Discord — all free) and logs to OpenSearch.
- Load testing with k6/Locust: simulate concurrent chat sessions, ramp virtual users, export
  results to Grafana (k6 supports Prometheus remote-write). Run as an on-demand or monthly Job,
  not always-on, since it's resource-intensive on a laptop.

## 7. Observability detail

FastAPI is instrumented with the OpenTelemetry SDK (traces + custom spans for "agent
decision", "DB query", "LLM call", "cache hit/miss"). The OTel Collector receives OTLP and fans
out to Jaeger (traces), Prometheus (metrics), and OpenSearch (logs). Grafana is the single-pane
dashboard on top of Prometheus, Jaeger, and OpenSearch as data sources.

## 8. Local Mac resource notes

- Run k3s inside a lightweight VM (Colima or Multipass) rather than Docker Desktop's K8s.
- Set conservative resource requests/limits on every pod; Postgres/Qdrant/RabbitMQ/OpenSearch
  are the heaviest — run OpenSearch single-node with a small heap (`-Xms512m -Xmx512m`).
- Use quantized GGUF models in Ollama (Q4_K_M) to keep LLM RAM/CPU usage manageable.
- You will not run ELK/OpenSearch, load tests, and the full data pipeline simultaneously on a
  single Mac comfortably — toggle namespaces on/off with `kubectl scale` or by pausing the
  ArgoCD app as needed.

## 9. Roadmap

See the "Roadmap" table in [README.md](../README.md) for the full 10-phase, 1-year plan.
