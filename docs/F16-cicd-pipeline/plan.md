# Implementation Plan: F16. CI/CD Pipeline

**Prerequisites:**
- F01 Project Foundation implemented: root workspace and `scripts/run-gates.mjs`, backend lint/typecheck/test scripts, Vitest projects with coverage, dependency-cruiser and knip configs, `.env.test`, the per-file test database helper, `docker/Dockerfile`, and the test and e2e compose files with WireMock
- Docker Engine with Compose v2 and buildx on the host; Node.js 24 and npm 10+
- New dev dependencies: `msw` 2.x, `testcontainers`, `@testcontainers/postgresql`
- Docker images: `postgres:17`, `rhysd/actionlint:1.7.7`, `registry:2` (local publish verification)
- GitHub repository `pedromeneghel/creator-insights-backend` with Actions enabled and GHCR available to `GITHUB_TOKEN`; a repository admin with an authenticated `gh` CLI, to apply branch protection once

### Stage 1: Local Job Entry Points

**1. Root CI scripts** - Add one root npm script per pipeline job, so lint, typecheck, architecture, unit, integration, e2e and the required-check aggregation can each be run with a single command, locally or in CI. The architecture script delegates to the provided gate runner.

**2. Lint job content** - Extend the backend with a Prettier check, its ignore file, and the fixture-anonymization check with its scanning rules, and chain them after ESLint in the lint entry point. Add workflow linting with actionlint.

**3. Coverage gate** - Add the branch threshold next to the existing line threshold for the domain and application globs, and add the coverage script that the unit entry point runs. Emit the summary report the CI job uploads.

### Stage 2: Test Isolation

**4. Network guard** - Add the MSW-based guard as a setup file for every Vitest project, with its loopback-host helper and the empty handler registry that later features extend with recorded YouTube and Telegram handlers.

**5. Testcontainers for integration** - Add the integration global setup, which starts a PostgreSQL 17 container when enabled and provides its admin URL. Update the per-file test database helper to prefer the provided URL over the `.env.test` value.

**6. Host-level egress blocking** - Create the shared blocked-hosts list and the idempotent runner script that black-holes those hosts, and add matching host mappings to the test and e2e compose services. Let the e2e compose reuse a prebuilt backend image.

**7. E2E runner** - Implement the e2e entry-point script: bring up the e2e stack, run the e2e project, dump service logs on failure, always tear down, and return the test command's exit code.

### Stage 3: Image Build and Merge Gate

**8. Image build and publish scripts** - Implement the buildx wrapper with optional layer cache and load-or-push output, and the publish script that validates its inputs, normalizes the owner, and pushes the commit-SHA and `latest` tags in one invocation with the revision label.

**9. Required-check aggregator** - Implement the aggregator CLI and its pure evaluation logic. It reads the workflow's needs context and fails, naming every unsuccessful or missing job.

### Stage 4: GitHub Workflow and Repository Settings

**10. CI workflow** - Write the workflow with its triggers, concurrency, least-privilege permissions, the six check jobs (each calling its root script, with egress blocking before tests), the always-running aggregate job, and the main-only publish job with GHCR login and the GitHub Actions layer cache. Register the new scripts and setup files in knip.

**11. Branch protection as code** - Add the script that configures `main` to require the aggregate check, strict status checks, and pull requests, with admins included, and reports API failures with the HTTP status and message.

**12. Rollout on GitHub** - Merge the workflow through a pull request, apply the branch protection script once as a repository admin, and confirm the first green push to `main` publishes both image tags to GHCR within the duration target.
