# Simmerr Backend (`simmerr-backend`)

> **Self-hosted AI meal planning that actually uses what you buy.**

Simmerr is an open-source, contracts-first backend that generates weekly meal plans, builds grocery carts, estimates nutrition, and exports recipes to your favorite tools. This repository contains the **Python (FastAPI) backend** for Simmerr v1.

---

## ✨ Key features (v1)

- **Contracts-first API** using Pydantic models → generates **OpenAPI** → **TypeScript types/SDKs** (consumed by `simmerr-frontend` and community tools).
- **Weekly meal planning** honoring diets, allergies, budgets, and skill goals (e.g., “learn to wok”).
- **Smart ingredient reuse** to minimize waste and optimize spending.
- **Recipe macros**: AI-assisted nutrition estimation with caching and confidence scoring.
- **Exports**: Mealie/Cronometer formats + generic CSV/JSON for shopping lists and plans.
- **Grocery integrations** via adapters (manual shopping list in v1; vendor order placement behind feature flags).
- **Background jobs & scheduling** for plan generation, validation, macro computation, cart building.
- **Self-host friendly** (Redis + Postgres), no cloud dependencies required.
- **Telemetry off by default**; opt-in, anonymous counters only.

---

## 🧱 Architecture overview

Simmerr favors clear seams, minimal magic, and replaceable parts.

```
simmerr-backend/
  packages/
    sm_ai/         # AI providers abstraction (LLMs, embeddings, nutrition lookups)
    sm_cli/        # CLI & admin tools (pack validators, seeds, maintenance)
    sm_config/     # Config & secrets (pydantic-settings), env schema
    sm_contracts/  # Pydantic models (SSOT for API types)
    sm_data/       # SQLAlchemy models, Alembic migrations, repositories
    sm_domain/     # Domain types/value objects (budgets, units, schedules)
    sm_services/   # Business logic (planning, macros, validation, carting)
    sm_vendor/     # Grocery adapters (manual, walmart, amazon_fresh, …)
  services/
    sm_api/        # FastAPI app, routers use sm_contracts exclusively
    sm_worker/     # Worker entrypoint (Taskiq or yaarq adapter) for background jobs
    sm_scheduler/  # APScheduler entrypoint (weekly jobs; Sunday 08:00 local)
```

**Service boundaries**

- `/ai`: tagging, macro estimation, plan synthesis (safe wrappers around providers).
- `/catalog`: ingredients, tools, cuisines, community **data packs** import.
- `/recipes`: CRUD, validation, macro estimation.
- `/plans`: generate/preview/approve weekly plans.
- `/orders`: cart build + order placement (feature-flagged; manual mode in v1).
- `/pantry`: track what you already have
- `/users`: auth identities, households, preferences, diets/allergies.

**Data highlights**

- Canonical **ingredients** with rich tags; **recipes** + ingredients + steps; **meal_plans** + items; **shopping_lists**; **orders**.
- **Ratings** feed back into planning preferences.
- Vendor products are mapped to ingredients when available (cached SKU search).

---

## 🔌 API and SDKs

- **OpenAPI** served at `GET /openapi.json` and interactive docs at `/docs` once running.
- **TypeScript types / clients** are generated in `simmerr-frontend` from this OpenAPI (see that repo for codegen specifics).
- Multi-language SDKs can be generated via OpenAPI Generator—templates are maintained in the community repo.

> **Note:** The API schemas live in **`sm_contracts`** and are the single source of truth for both backend validation and frontend/client types.

---

## 🗓️ Background jobs & scheduler

- **Jobs**: recipe macro computation, recipe validation, weekly plan generation, cart build, exports.
- **Queue**: pluggable adapter (`Taskiq + Redis` in v1). A thin interface allows swapping to **`yaarq`** later with no call-site changes.
- **Scheduler**: APScheduler runs household-scoped weekly planning (default Sunday 08:00 local, configurable).

---

## 🧰 Community data packs

Simmerr supports importing community-curated packs of **ingredients**, **recipes**, and **tags**.

- **Format**: versioned tarball with `pack.json` (manifest), `ingredients.jsonl`, `recipes.jsonl`, `tags.jsonl`, optional `media/`.
- **Validation**: `sm_cli` provides `validate-pack` and `import-pack`.
- **Conflict policy**: canonical IDs (slugs) with safe upserts; `--force` to override.

> For pack authoring guides and contributed packs, see the `simmerr-community` repository.

---

## 🛒 Vendor adapters

- **Manual** (v1): build vendor agnostic shopping lists
- **Walmart**: alpha adapter behind a feature flag.
- **Amazon Fresh**: alpha adapter behind a feature flag.

Adapters implement a common interface:

- `searchProducts`
- `priceEstimate`
- `buildCart`
- `placeOrder` (optional; off by default)

---

## 🔐 Security, privacy, and compliance

- **Secrets & config** via `sm_config`; typed env schema; safe defaults.
- **At-rest encryption** for vendor credentials (AES-GCM per household key).
- **Rate limiting** on sensitive endpoints (plan generation, ordering).
- **PII handling**: only the minimum necessary; see `PRIVACY.md` (TODO).
- **Telemetry**: disabled by default; when enabled, anonymous counters only.

---

## 🚀 Quick start

> Below is the high-level flow once you’ve completed bootstrapping.

1. **Run migrations**
   `alembic upgrade head`

2. **Start API & workers**

   - API: `uv run services/sm_api/main.py` (or via your process manager)
   - Worker: `uv run services/sm_worker/worker.py`
   - Scheduler: `uv run services/sm_scheduler/entrypoint.py`

3. **Create a household & prefs (example)**

   - `POST /users` → create user
   - `POST /households` → create household
   - `POST /preferences` → diets/allergies/budget/tool goals

4. **Import a starter pack**

   - `POST /catalog/packs/import` (multipart upload)

5. **Generate & approve plan**

   - `POST /plans/generate`
   - `POST /plans/{id}/approve`

6. **Preview (or place) order**

   - `POST /orders/preview` (manual mode OK)
   - `POST /orders/place` (if adapter + creds configured and feature enabled)

7. **Export to tools**

   - `POST /recipes/{id}/export`

---

## ⚙️ Configuration

All configuration is environment-driven via `sm_config` and documented in **`BOOTSTRAP.md`**:

- Database, Redis, queue adapter, scheduler cadence
- AI provider keys
- Feature flags (vendor adapters, order placement)
- Telemetry/observability (OpenTelemetry hooks)

---

## 🧪 Testing & quality

- **Tests**: `pytest` + `httpx` for API; factories for data; integration tests use ephemeral DB/Redis.
- **Types**: `pyright` (strict).
- **Lint**: `ruff`.
- **Pre-commit** hooks are provided.

---

## 📦 Releases & images

- Semantic Versioning (**SemVer**) for packages and containers.
- Multi-arch Docker images (linux/amd64, linux/arm64) are published from CI.

  - `simmerr/api:<version>`
  - `simmerr/worker:<version>`
  - `simmerr/scheduler:<version>`

Release notes are generated automatically and attached to GitHub Releases.

---

## 🧭 Roadmap

- Better pantry modeling & depletion tracking (leftovers, perishables windows)
- Planner ILP/OR mode for hard budgets & waste minimization
- Adapter ecosystem (Instacart/regionals), richer SKU mapping
- First-class nutrition DB integration (fallbacks + caching)
- Admin UI for DLQ/retries/metrics (worker dashboard)
- Signed community packs + registry

> For a living roadmap and community discussion, see the `simmerr-community` repo.

---

## 🤝 Contributing

We welcome issues and PRs! Please read:

- **`CONTRIBUTING.md`** (TODO) for code standards and workflow
- **`AGENTS.md`** (repo-specific conventions for AI usage, scaffolding, and codegen)
- **`CODE_OF_CONDUCT.md`** (TODO)

---

## 📚 Documentation

- **API reference**: `/docs` once running
- **Community packs & self-host Compose**: `simmerr-community` repo
- **Frontend / SDK usage**: `simmerr-frontend` repo

---

## 📝 License

**TODO**: Choose a license (Apache-2.0 or MIT recommended for OSS + SaaS friendliness).

---

## 🧩 FAQ

- **Is order placement required?**
  No. Manual cart preview works without vendor creds.

- **Can I run this fully offline?**
  Yes for core features (planning, packs, exports). Live pricing and order placement need network access.

- **Does it support ARM (Raspberry Pi)?**
  Yes—multi-arch images are provided.

---

## 🏷️ Status

- **Simmerr v1**: API stable for core flows; vendor adapters are evolving behind feature flags.
- Expect additive, backwards-compatible changes to contracts; breaking changes will bump the major version.

---

Made with ❤️ by the Simmerr community.
