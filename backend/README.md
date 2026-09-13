# backend

FastAPI application, agent orchestrator, and tool definitions.

- `app/` — FastAPI routes, request/response models, startup wiring
- `agent/` — LangGraph/LlamaIndex agent orchestration logic
- `tools/` — agent tools (SQL query, vector search, cache lookup)

Talks to Postgres, Qdrant, and Redis directly (read side); calls the local Ollama LLM for
generation. Uses schemas from `shared/`.
