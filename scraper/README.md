# scraper

Scrapy/Playwright spiders, the normalizer, and the embedding worker.

- `spiders/` — one spider per source site (or source category)
- `normalizer/` — dedup + schema validation, writes to Postgres and `data/`
- `embedder/` — chunks markdown records and upserts embeddings into Qdrant

Publishes to and consumes from RabbitMQ — see `docs/architecture.md` section 2 (Data flow).
Uses schemas from `shared/`.
