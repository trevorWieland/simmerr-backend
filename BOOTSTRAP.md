# BOOTSTRAP — `simmerr-backend` (v1)

> **Purpose:** Temporary, repo-internal knowledge base to get **v1** stood up, tested, linted, typed, containerized, and shipping CI/CD.
> When v1 is stable and docs are split, **delete this file** and migrate content to: `README.md`, `INSTALLATION.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `PRIVACY.md`, `AGENTS.md`, and ADRs.

---

## 1) Vision & scope (v1)

Simmerr is a **self-hostable, contracts-first** backend for AI-powered meal planning that produces weekly plans, reuses ingredients to minimize waste, estimates nutrition, and (optionally) builds grocery carts via vendor adapters. The **frontend and community repos** will consume the OpenAPI contract generated here.

**v1 goals**

- Ship a stable **OpenAPI** contract and a minimal but complete **core flow**:

  1. Import community data packs (ingredients, recipes, tags)
  2. Generate a weekly plan honoring prefs (diet, allergies, budget, goals)
  3. Approve a plan and produce a shopping list
  4. Preview a vendor cart (manual mode OK)
  5. Estimate macros per recipe; export to Mealie/Cronometer

- Background jobs + weekly scheduler (Sunday 08:00 local default)
- Contracts-first model; **TypeScript types** generated downstream (frontend)
- Self-host friendly defaults (Postgres, Redis)
- Production-grade CI: format, lint, type-check, test, build multi-arch images, publish OpenAPI artifact

Out-of-scope (v1): mobile apps, full pantry depletion logic, rich admin UI, multi-tenant billing, dashboards for queues (can be CLI-first).

---

## 2) Architecture & repo layout

**Guiding principles**

- **Contracts-first**: `sm_contracts` is SSOT for API I/O.
- **Replaceable seams**: vendors, queue adapter, AI providers.
- **Minimal magic**: explicit services over framework entanglement.
- **OSS-first ergonomics**: multi-arch images, env-driven, low resource use.

```
simmerr-backend/
  packages/
    sm_ai/         # AI providers abstraction (LLMs, nutrition mapping, embeddings)
    sm_cli/        # CLI tools: validate/import packs, seed, maintenance
    sm_config/     # Typed settings (pydantic-settings), env schema, feature flags
    sm_contracts/  # Pydantic v2 models (SSOT): request/response, domain DTOs
    sm_data/       # SQLAlchemy 2.0 models, Alembic migrations, repositories
    sm_domain/     # Domain types/value objects (budgets, units, scoring)
    sm_services/   # Business logic (planning, macros, validation, carting)
    sm_vendor/     # Vendor adapters (manual, walmart, amazon_fresh)
  services/
    sm_api/        # FastAPI app; routers use sm_contracts exclusively
    sm_worker/     # Background worker (Taskiq adapter initially)
    sm_scheduler/  # APScheduler entrypoint for weekly jobs
  .github/
    workflows/     # CI pipelines (lint, type, test, build, publish)
  docs/
    adr/           # Architecture Decision Records (see §11)
```

**Python**: 3.13 (target 3.14 when available), **uv** for package management, **ruff** (lint+fmt), **pyright** (types).

---

## 3) Tech stack

- **API**: FastAPI + Uvicorn, Pydantic v2 (strict mode), orjson
- **DB**: PostgreSQL 16, SQLAlchemy 2.0 (async), Alembic, Full ENV controls (DB_HOST, DB_PORT, etc.)
- **Cache/Queue**: Redis 7
- **Jobs**: Taskiq + `taskiq-redis` (pluggable; `yaarq` adapter later)
- **Scheduler**: APScheduler (cron-like, runs in its own service)
- **Units & nutrition**: `pint` for units; nutrition via AI + cached lookups
- **Contracts**: OpenAPI generated from `sm_api`, schemas sourced from `sm_contracts`
- **Testing**: pytest, httpx, pytest-asyncio, coverage, testcontainers
- **Quality**: ruff (lint/format), pyright (strict), pre-commit
- **Containers**: multi-arch Docker builds (amd64/arm64)
- **Observability**: structured JSON logs; OTel hooks (TODO enable by flag)

---

## 4) Contracts-first API (SSOT)

- **All** request/response models live in `packages/sm_contracts`.
- Routers in `sm_api` import only from `sm_contracts` (not from persistence models).
- **OpenAPI** served at `/openapi.json`; docs at `/docs`.
- **API Versioning**: prefix routes with `/api/v1/...` for stability.
- **Error shape**: use RFC 7807 Problem Details (`application/problem+json`) with `type`, `title`, `status`, `detail`, `instance`.

**Compatibility policy (v1)**

- Additive changes favored.
- Breaking changes require a major bump (v2) and deprecation window.

---

## 5) Domain model (baseline)

- **Users, Households, AuthIdentities**
- **Preferences** (diet, allergies, budget, cuisines, disliked ingredients)
- **Ingredients** (canonical + tags), **VendorProducts** (SKU mapping)
- **Recipes** (steps, ingredients with qty/unit), **RecipeMacros**
- **MealPlans** (weekly), **MealPlanItems** (per day/meal)
- **ShoppingLists** + items
- **Orders** + items (mapped from vendor products)
- **Ratings** (feedback loop)
- **PantryItems**

**Migrations**: Alembic; 1 migration per PR that changes schema; never edit historical migrations.

---

## 6) Core services & endpoints (v1)

**Catalog & Packs**

- `POST /api/v1/catalog/packs/import` (multipart); validate and staged upsert
- `GET /api/v1/catalog/ingredients` (filter by tag)

**Recipes**

- CRUD
- `POST /api/v1/recipes/{id}/macros/estimate`
- `POST /api/v1/recipes/{id}/export?format=mealie|cronometer`

**Planning**

- `POST /api/v1/plans/generate` (householdId, weekOf, goals, budget, toolFocus)
- `POST /api/v1/plans/{id}/approve`
- `GET  /api/v1/plans/{id}`

**Orders**

- `POST /api/v1/orders/preview` (manual mode OK)
- `POST /api/v1/orders/place` (feature-flagged; per-vendor)

**Users/Prefs**

- `POST /api/v1/users`, `POST /api/v1/households`
- `POST /api/v1/preferences` (by household)

**Health**

- `GET /health` (API/DB/Redis checks)

> Non-goals: auth provider integration in v1 (use PAT/JWT stub if needed for CI). Real auth deferred to frontend/SaaS integration, and we'll want OIDC for

---

## 7) Background jobs & scheduler

**Queue adapter interface** lives in `sm_services.queue`. Default: Taskiq Redis.
Tasks:

- `compute_recipe_macros(recipe_id)`
- `validate_recipe(recipe_id)`
- `generate_weekly_plan(household_id, week_of)`
- `build_vendor_cart(plan_id, vendor)`
- `place_vendor_order(order_id)`
- `export_recipe(recipe_id, format)`

**Scheduler** (`sm_scheduler`):

- Household-scoped weekly plan generation (default **Sunday 08:00** local; configurable via env).
- Use idempotency guards to avoid duplicates.

---

## 8) Vendor adapters

- **manual** (v1, default): only build/preview carts, no external APIs.
- **walmart** (alpha, feature-flagged)
- **amazon_fresh** (alpha, feature-flagged)

Common interface in `sm_vendor.base`:

- `search_products(query | ingredient_slug)`
- `price_estimate(items)`
- `build_cart(items)`
- `place_order(cart, payment_token)` (optional; feature-flagged)

---

## 9) Configuration, feature flags, secrets

Centralized in `sm_config` (pydantic-settings). Provide **typed** settings classes and a machine-readable **env schema**.

**Flag categories**

- `FEATURE_VENDOR_WALMART`, `FEATURE_VENDOR_AMAZON_FRESH`
- `FEATURE_ORDER_PLACEMENT` (off by default)
- `FEATURE_TELEMETRY` (off by default)
- `SCHEDULER_ENABLED`

**Secrets**

- DB/Redis URLs, AI provider keys, per-household vendor creds (encrypted at rest).

> Actual variable names, defaults, and examples move to `INSTALLATION.md`. Keep this file repo-focused.

---

## 10) Security & privacy baseline

- **At-rest encryption** (AES-GCM) for vendor credentials with per-household keys.
- **Idempotency**: support `Idempotency-Key` header on plan/order endpoints.
- **Rate limiting** on plan generation & order endpoints (basic token-bucket; per-IP or per-household).
- **PII minimization**: store only what's required.
- **Audit logging** (structured JSON) for sensitive ops (plan approve, order place).
- **Telemetry**: off by default; when enabled, anonymous counters only.
- **Vulnerabilities**: weekly dependency audit in CI (pip-audit or uv audit) + container image scans.

(TODO move full details to `SECURITY.md` and `PRIVACY.md`.)

---

## 11) Decision records (ADR)

Create **Architecture Decision Records** in `docs/adr/` using a simple template:

```
# ADR-0001: Title
Date: YYYY-MM-DD
Status: Proposed | Accepted | Superseded by ADR-00XX
Context:
Decision:
Consequences:
Alternatives considered:
```

Initial ADRs to capture:

- ADR-0001: Contracts-first models in `sm_contracts`
- ADR-0002: Taskiq as initial queue adapter (pluggable, `yaarq` candidate)
- ADR-0003: RFC7807 for error payloads
- ADR-0004: API versioning strategy `/api/v1`
- ADR-0005: Manual vendor adapter default; feature flags for order placement

---

## 12) Coding standards & style

**Python**

- **Ruff** as linter and formatter (`line-length: 100` unless overridden).
- **Pyright** strict mode; no implicit `Any`.
- Require Pydantic models for DTOs; never pass raw dicts across service boundaries.
- Explicit imports (no wildcard).
- Async-first I/O (httpx, async db session).

**Schemas**

- All public I/O goes through `sm_contracts`. No persistence models in responses.
- API responses are **snake_case** for JSON (or define a casing policy and stick to it).

**Services**

- Keep pure business logic in `sm_services`. Routers thin.
- Use **value objects** in `sm_domain` for units, budgets, dates.

**Migrations**

- One Alembic migration per schema-affecting PR. No squash in v1.

**Docs**

- Update ADRs when decisions change.
- Short module-level docstrings for non-obvious modules.

---

## 13) Testing strategy

**Test layers**

- **Unit**: services, domain, helpers (fast, pure).
- **Contract**: schema-level tests to ensure OpenAPI matches `sm_contracts` (drift check).
- **Integration**: API with ephemeral Postgres/Redis (containers in CI).
- **Worker**: enqueue/execute tasks; DLQ and retry behavior.
- **Planner**: seed datasets → assert constraints (diet/allergy/budget/tool goals) satisfied.

**Quality gates (CI)**

- Ruff (lint+fmt) → Pyright (types) → Pytest (unit+integration) → Coverage ≥ **80%** (raise later).
- OpenAPI drift check must pass: routers ↔ `sm_contracts`.

(TODO move commands to `CONTRIBUTING.md`.)

---

## 14) CI/CD overview

**Branching**

- Trunk-based: `main` is protected (required checks).
- Feature branches: `feat/...`, `fix/...`, `chore/...`.
- Conventional Commits required; semantic-release for versioning.

**Workflows** (GitHub Actions)

- `ci.yml`: lint, type, test, build OpenAPI artifact.
- `build-images.yml`: build & push multi-arch images on tags.
- `security.yml`: dependency & image scans on a schedule.
- `release.yml`: semantic-release (generates GitHub Release notes).
- `docs.yml` (TODO): publish API reference snapshot if needed.

**Artifacts**

- `openapi.json` uploaded on each successful `main` build.
- Docker image tags: `api:v1.x.y`, `worker:v1.x.y`, `scheduler:v1.x.y` + `latest`.

_(Exact YAML goes to `.github/workflows/`, keep this section descriptive.)_

---

## 15) Release/versioning policy

- **SemVer** per service image and repo tag.
- Additive API changes: patch/minor. Breaking: major + `/api/v2` if necessary.
- Changelogs auto-generated by semantic-release.

---

## 16) Observability

- **Logging**: JSON logs with contextual fields (service, request_id, user/household_id, task_id).
- **Tracing**: OTel instrumentation hooks (disabled by default; enable via env).
- **Metrics**: minimal counters/histograms (requests, queue timings, planner scores). (TODO export)

Move details to `OBSERVABILITY.md` later.

---

## 17) Community data packs

- Format: `.tar.gz` with `pack.json` (name, version, license, checksum), `ingredients.jsonl`, `recipes.jsonl`, `tags.jsonl`, optional `media/`.
- Import path: `sm_cli` (validate) → API import endpoint (staged upsert).
- Conflict policy: canonical slugs; prefer canonical unless `--force`.
- Signing/registry: TODO (post-v1).

Move authoring guide to `simmerr-community`.

---

## 18) Security reviews & threat model (brief)

- **Secrets**: centralize in `sm_config`; prefer process env / Docker secrets.
- **Credentials**: per-household vendor creds encrypted; key rotation policy (TODO).
- **Idempotency**: protect plan generation & order placement.
- **Rate limiting**: coarse-grained for v1; fine-grained later.
- **Data retention**: define TTLs for task results/logs (TODO).
- **Audit**: record plan approvals and order placements with opaque IDs.

Formalize in `SECURITY.md` post-v1.

---

## 19) Open questions / TODO buckets

- Choose **license** (MIT or Apache-2.0 recommended).
- Finalize JSON casing policy (snake_case vs camelCase).
- Nutrition db fallback strategy & caching policy.
- Vendor adapter sandboxing and contract tests.
- Observability exporters (Prometheus, OTLP).
- Auth story for OSS self-host (PAT/JWT vs OIDC) — likely frontend concern.
- CLI UX for DLQ/metrics until an admin UI exists.

---

## 20) v1 Definition of Done (DoD)

Simmerr v1 is **done** when all of the following are true:

### Contracts & API

- [ ] All v1 endpoints implemented and documented under `/api/v1/*`.
- [ ] `sm_contracts` is authoritative; OpenAPI generated with no validation errors.
- [ ] RFC7807 errors implemented for failure paths.

### Jobs & Scheduler

- [ ] Queue adapter (Taskiq) wired; tasks have retries, backoff, DLQ.
- [ ] Weekly scheduler runs and is idempotent.

### Data & Packs

- [ ] Alembic migrations for baseline schema.
- [ ] Pack validation/import works with example starter pack.

### Quality gates

- [ ] Ruff lint + format passes.
- [ ] Full docstrings linting, Field usage, consistency.
- [ ] Pyright strict passes (no implicit Any).
- [ ] Tests: unit + integration pass in CI; coverage ≥ 80%.
- [ ] OpenAPI drift check passes.

### Security & config

- [ ] Secrets via `sm_config`; feature flags respected.
- [ ] Vendor creds encrypted at rest.
- [ ] Basic rate limiting on plan/order endpoints.

### CI/CD & release

- [ ] CI builds and uploads `openapi.json` artifact on `main`.
- [ ] Multi-arch Docker images build on tag and push to registry.
- [ ] Semantic-release generates changelog and tags.

### Docs & hygiene

- [ ] `README.md` reflects v1 features.
- [ ] `INSTALLATION.md` exists for local/dev setup (envs, compose).
- [ ] `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` initial versions added.
- [ ] At least 3 ADRs added and accepted.

When the above are met, **migrate content** from this BOOTSTRAP to the permanent docs and **delete this file**.

---

## 21) Appendix — Naming & conventions

- Python package prefixes: `sm_` (e.g., `sm_contracts`, `sm_services`).
- Module names: short, descriptive, snake_case.
- Commit style: **Conventional Commits** (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`).
- Branch names: `feat/<short>`, `fix/<short>`, `chore/<short>`.
- Issue/PR templates: TODO (create in `.github/`).

---

**End of BOOTSTRAP.md** — this file exists to accelerate v1.
Delete after v1 ships and move content into the permanent docs.
