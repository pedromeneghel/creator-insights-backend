# Implementation Plan: F01. Project Foundation

**Prerequisites:**
- Docker Engine with Compose v2 (the only host requirement for running the stack)
- Node.js 24 LTS and npm 10+ on the host for authoring, linting, and unit tests
- Base images: `node:24-slim`, `nginx:alpine`, `postgres:17`, `wiremock/wiremock`
- Environment variables documented in `.env.example` (see spec §3 Config scope); a 32-byte base64 `ENCRYPTION_KEY` for local runs
- The clean-arch skill templates in `.claude/skills/clean-arch/templates/scaffold/` as the starting point for root scripts and gate configs

### Stage 1: Workspace and Tooling

**1. Monorepo root** - Create the npm workspace root with the `apps/backend` workspace, the root scripts that delegate to the backend, and the git-ignore rules. Copy `run-gates.mjs` from the clean-arch scaffold unchanged.

**2. Backend package and TypeScript setup** - Create the backend package with Node 24, TypeScript 6 strict ESM settings, per-layer path aliases, and separate typecheck and build configurations. Add the runtime and dev dependencies listed in the spec's decisions.

**3. Lint, format, and test runner** - Configure ESLint flat config and Prettier, including the restrictions on environment access and infrastructure instantiation. Configure Vitest with unit, integration, and e2e projects and the coverage thresholds for the domain and application layers.

**4. Architecture gates retargeted to the PRD layout** - Bring in the dependency-cruiser config, the architecture check script, and the knip config from the clean-arch scaffold, and change only their path patterns to match the PRD layer layout. Confirm `run-gates.mjs` runs clean against the empty skeleton.

**5. Agent instructions update** - Update CLAUDE.md and AGENTS.md so the backend layer and composition-root wording matches the PRD layout chosen for this project.

### Stage 2: Core Layers and Configuration

**6. Configuration module** - Implement the Zod-validated configuration as the single reader of the process environment. It should expose a pure parse function and the frozen config, and invalid input should produce messages that name each variable and its expected shape without leaking secret values.

**7. Domain shared kernel** - Add the error hierarchy carrying the PRD error codes and HTTP statuses, the typed ID value object, the pagination types, and the unit-of-work interface.

**8. Application ports and health use case** - Define the clock, cipher, database-health, settings-store, and logger ports, and the health use case with its DTOs.

**9. Clock, cipher, and logger adapters** - Implement the system clock, the AES-256-GCM cipher with the versioned envelope format and its failure modes, and the pino logger factory with redaction paths and a boot-level correlation ID.

### Stage 3: Persistence

**10. Database connection** - Implement the pg pool, the Drizzle client, and the connect-with-retry routine with the PRD's attempt count, backoff, and failure message.

**11. Schema and first migration** - Define the settings and migrations-tracking tables in the Drizzle schema module, configure drizzle-kit, and generate the first migration with its journal.

**12. Migration runner** - Implement the lock-protected migration runner that applies each pending journal entry in its own transaction, records name and hash, rejects edited migrations, and reports the failing migration by name.

**13. Settings store and database health probe** - Implement the settings store adapter over the settings table and the database health probe with its 2-second timeout.

### Stage 4: HTTP Interface and Composition Root

**14. HTTP server foundation** - Build the Fastify server: Zod type provider, correlation-ID handling, logger injection, the central error-envelope mapping, the not-found handler, and route registration under the versioned prefix.

**15. OpenAPI publication** - Generate the OpenAPI 3.1 document from the route schemas and serve it as JSON and as the Swagger UI under the docs path, excluding hidden routes.

**16. Health and diagnostics routes** - Add the health handler and route with 200/503 behavior, plus the hidden diagnostics routes, which are mounted only when the test-routes flag is enabled.

**17. Awilix container** - Register config, logger, clock, pool, Drizzle client, cipher, settings store, health probe, use cases, handlers, and routes in the container, which stays the only place concrete infrastructure is created.

**18. Role entrypoints and healthcheck** - Implement the role dispatcher, the API boot (connect, migrate, listen), the minimal worker boot with its ready marker, graceful shutdown for both roles, and the role-aware healthcheck script.

### Stage 5: Containers and Developer Workflow

**19. Backend Dockerfile** - Create the multi-stage image with dependency, build, production-dependency, and runtime stages. The runtime stage runs as a non-root user on the slim Node base, with the healthcheck and role-selecting entrypoint.

**20. Stub web image** - Create the nginx stub with a placeholder page, a health path, and the `/api` proxy to the api service that preserves the correlation header.

**21. Local production compose** - Define web, api, worker, and db with health checks, dependency ordering, restart limits, the database volume, the `WEB_IMAGE` override, and a single host port bound to loopback 8080.

**22. Dev, test, and e2e compose files** - Create the hot-reload dev topology, the containerized test topology that runs every quality step, and the e2e topology with WireMock and an empty mappings folder.

**23. Environment template and dev scripts** - Write `.env.example` documenting every PRD variable, the backend `.env.test`, and the `init.sh`, `stop.sh`, and `resolve-env.sh` scripts that drive the dev compose with per-worktree project names and free-port selection.
