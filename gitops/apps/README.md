# gitops/apps

One subfolder per ArgoCD child Application (Helm chart or Kustomize overlay), discovered by
the root app-of-apps Application in `gitops/root-app.yaml`.

| Folder | Deploys |
|---|---|
| `backend/` | FastAPI + agent service |
| `scraper/` | Scraper Deployment/CronJob, normalizer, embedder workers |
| `data-layer/` | Postgres, Redis, RabbitMQ, Qdrant |
| `observability/` | OpenSearch, Dashboards, OTel Collector, Jaeger, Prometheus, Grafana |
| `mesh/` | Linkerd config, NetworkPolicies |
| `testing/` | k6 load test Jobs, availability-check CronJob |

Populate these starting Phase 4/5 of the roadmap — see CLAUDE.md.
