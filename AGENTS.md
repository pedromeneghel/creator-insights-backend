## Monorepo
- apps/web/ - Frontend / Next.js 16
- apps/backend/

## Environment
- Use `./scripts/init.sh` to start the project.
- Use `./scripts/stop.sh` to stop dev processes.
- Use `./scripts/stop.sh --clean` to stop processes and reset generated local state. It drops the branch database when DB support is enabled.
- Do not run app dev servers directly.
- Before starting any task, init the Environment.


## Backend
- Before backend work, you must load/use the `clean-arch` skill.
- Follow clean architecture: `domain/` <- `usecase/` <- `infra/`.
- `src/main.ts` is the composition root and the only place that wires concrete infra classes.
- `process.env` is only allowed in `src/config/env.ts`.
- Handlers validate input and call use cases; handlers do not catch errors.
- Run `./scripts/run-gates.mjs` before declaring backend work done.
- Use the provided gates and linters. Do not create replacement gates.

## Code Style
- Keep functions small, focused, and explicitly typed.
- Prefer small modules over large files; Follow SOLID (SRP), a component must have only one reason for change
- Avoid `any`, duplicated logic, and deep nesting.
- Error messages must include the offending value and expected shape.
- Comments explain why, not what.
- Early returns over nested ifs. Max 2 levels of indentation.

## Backend tests And Logs
- Every new backend function gets a test; bug fixes get a regression test.
- Mock external I/O with named fake classes, not inline stubs.
- Use structured JSON for observability logs and plain text for CLI output.

## Frontend Tests
- Always use playwright-cli for any type of navigation test. Load playwright-cli skill. Do not use Playwright. Do not create e2e folders for playwright tests.
- Use the design system and pages: `docs/design/design-system-pages/` to confirm the UI matches the design.
- Do not create granular tests. Only critical ones that can change business rules or important behavier on the UI

## Design System and Pages
- Use the design system and pages: `docs/design/design-system-pages/`
- Use ONLY the Option B: "B · Signal" - Dark.
- Use as reference. The source does not cover problem

## Video samples for video upload
- Video 1: https://drive.google.com/file/d/1JM1jrv5CHJnyILDuqXpEkZB0MyQCLkSl/view?usp=sharing (94MB)
- Video 2: https://drive.google.com/file/d/1r4ovc2PiLMUt7IHf2k7IPR1p7GIfeOm6/view?usp=sharing (454MB)
- Save the videos on `video-samples` as needed.

## OpenAI API Key
- Get it from system env

## Rules
- NEVER create temp files in the project root, such as playwright tests files, scripts.