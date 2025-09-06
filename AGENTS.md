# AGENTS.md — Guidance for AI Assistants and Contributors

Scope: This file applies to the entire repository. If more specific AGENTS.md files are added in subdirectories later, those take precedence for files in their subtree.

Purpose: Give clear, stable instructions so assistants can make safe, high‑quality changes that align with Simmerr’s contracts‑first architecture and v1 bootstrap plan.

## Core Principles
- Contracts‑first: Public I/O types live in `packages/sm_contracts` (Pydantic). Routers and services depend on these models; never return persistence models from APIs.
- Clear seams: Thin API routers. Business logic in `sm_services`. Domain value objects in `sm_domain`. Data access in `sm_data`.
- Replaceable parts: Pluggable queue, vendor adapters, and AI providers behind clear interfaces.
- Minimal magic: Prefer explicit, typed, testable code over hidden behavior.
- Self‑host friendly: Default stack is Postgres + Redis; no required cloud dependencies.

## Repository Layout (target v1)
See README.md for the overview. Expected structure once bootstrapped:

```
packages/
  sm_ai/         # AI provider abstractions
  sm_cli/        # CLI & admin tools (validators, seeds, ops)
  sm_config/     # Typed settings and feature flags
  sm_contracts/  # Pydantic models (SSOT for API types)
  sm_data/       # SQLAlchemy models, Alembic migrations, repos
  sm_domain/     # Value objects (budgets, units, schedules)
  sm_services/   # Business logic (planning, macros, carting)
  sm_vendor/     # Grocery adapters (manual, walmart, amazon_fresh, …)
services/
  sm_api/        # FastAPI app & routers (contracts-only I/O)
  sm_worker/     # Background worker entrypoint (Taskiq adapter)
  sm_scheduler/  # APScheduler entrypoint (weekly jobs)
```

## Editing Protocol (AI assistants)
1) Read first, then change
- Before modifying code, read `README.md`, `BOOTSTRAP.md`, and any AGENTS.md affecting files you’ll touch.
- Check for ADRs in `docs/adr/` if present; add/update ADRs when a design decision is introduced or changed.

2) Keep changes surgical
- Follow the existing style and structure. Don’t rename or reshuffle files unless required by the task.
- Prefer minimal patches that solve the root cause.

3) Contracts drive implementation
- Adding/changing an endpoint:
  - Define request/response models in `sm_contracts` first.
  - Implement router handlers in `services/sm_api` using only `sm_contracts` types.
  - Delegate to `sm_services` for logic; use `sm_domain` value objects as appropriate.
  - Errors must follow RFC7807 problem+json.

4) Data and migrations
- Add/alter SQLAlchemy models in `sm_data` and create a new Alembic migration for every schema change (no squashing in v1).
- Use repositories for DB access; avoid direct session calls in routers/services.

5) Background jobs & scheduler
- Queue tasks are declared in `sm_services` and executed by `services/sm_worker` via the queue adapter (Taskiq in v1).
- Scheduled jobs belong in `services/sm_scheduler` (weekly household planning; Sunday 08:00 local by default).

6) Vendor adapters
- Implement adapters under `sm_vendor/<name>/` by conforming to the common interface in `sm_vendor.base`:
  - `search_products`, `price_estimate`, `build_cart`, optional `place_order` (behind feature flags).
- The default v1 path is a manual adapter that builds shopping lists without external APIs.

7) CLI commands
- Add operational/admin commands under `sm_cli` (validation, imports, seeds, maintenance). Prefer simple, composable subcommands.

8) Configuration & feature flags
- Centralize in `sm_config` using pydantic‑settings. Keep flags typed and documented. Respect defaults that are safe for self‑hosting.

9) Security & privacy
- Secrets and vendor credentials flow through `sm_config` and are encrypted at rest (AES‑GCM per household key).
- Add idempotency on plan/order endpoints; apply basic rate limiting to sensitive routes.

10) Observability (off by default)
- Structured JSON logs with contextual fields. Wire tracing/metrics hooks but keep disabled unless explicitly enabled via env.

## Coding Standards
- Python: Ruff for lint+format (line‑length 100), Pyright strict typing. No wildcard imports. Async‑first I/O.
- DTOs: Always Pydantic models for public I/O. Don’t pass raw dicts across service boundaries.
- JSON casing: Use snake_case for now (until a policy change is adopted repo‑wide).
- Docstrings: Module‑level for non‑obvious modules; public functions/methods should have concise docstrings.

## Testing Strategy
- Unit: `sm_services`, `sm_domain`, helpers.
- Contract: Assert OpenAPI matches `sm_contracts` (drift check).
- Integration: API + ephemeral Postgres/Redis (containers in CI).
- Worker: Task enqueue/execute, retries, backoff, DLQ.
- Planner: Seeded datasets that assert diet/allergy/budget/tool constraints are satisfied.

CI quality gates (target): Ruff → Pyright → Pytest; coverage ≥ 80%; OpenAPI drift must pass.

## Making v1 Happen — Suggested Build Order
1) Scaffolding
- Create package layout shown above with minimal `pyproject.toml` per package and service entrypoints.
- Add `sm_contracts` with core primitives (ingredients, recipes, plans, shopping lists, RFC7807 errors).

2) API skeleton
- Wire `services/sm_api` FastAPI app; serve `/openapi.json`, `/docs`, `/health`.
- Add placeholder routers for `recipes`, `plans`, `orders`, `catalog`, `users` using `sm_contracts` types.

3) Data layer
- Set up `sm_data` with SQLAlchemy models and initial Alembic migration for baseline schema.

4) Services and jobs
- Implement core services for recipes, macro estimation, planning, and cart building.
- Add Taskiq queue adapter, worker entry, and APScheduler for weekly planning.

5) CLI and packs
- Implement `sm_cli` commands: `validate-pack`, `import-pack`.

6) Vendor
- Implement the manual adapter; gate other vendors behind feature flags.

7) Hardening
- Security baselines (idempotency, basic rate limits), observability hooks, tests, and CI workflows.

Use BOOTSTRAP.md’s “v1 Definition of Done” as acceptance criteria. Move/trim BOOTSTRAP.md content into permanent docs as v1 stabilizes.

## Do/Don’t (Safety)
- Do: Add ADRs for notable decisions; keep changes small; write/adjust tests with behavior changes; respect feature flags and defaults.
- Don’t: Introduce breaking API changes without updating `sm_contracts`, tests, and versioning notes; expose persistence models in APIs; enable telemetry by default; add heavy dependencies without justification.

## Housekeeping
- Commits: Conventional Commits (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`). Branches: `feat/...`, `fix/...`, `chore/...`.
- Migrations: One per schema‑affecting PR.
- Docs: Keep README accurate; add/update ADRs as decisions change.

## Open Questions (trackers)
Mirror BOOTSTRAP.md’s open questions until resolved: JSON casing policy, license, nutrition DB fallback, vendor adapter contract tests, observability exporters, auth story for OSS self‑host, CLI UX for DLQ/metrics.

— End AGENTS.md —
