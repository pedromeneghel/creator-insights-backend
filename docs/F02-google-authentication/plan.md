# Implementation Plan: F02. Google Authentication

**Prerequisites:**
- F01 Project Foundation implemented: config module, error envelope, route descriptor, Drizzle migrator, logger, Awilix container, compose files, WireMock in the e2e compose
- New dependencies: `@fastify/cookie`, `jose`
- Environment variables: `ALLOWED_EMAIL`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `SESSION_SECRET`, plus the defaulted URL variables listed in the spec's Assumptions
- A Google Cloud OAuth client (web application) whose authorized redirect URI is `http://localhost:8080/api/v1/auth/google/callback`, for real logins outside tests

### Stage 1: Domain and Persistence

**1. Configuration additions** - Extend the configuration module with the allowed email, Google client credentials, session secret, dashboard URL, and the overridable Google endpoint URLs and issuers, validated the same way as F01's variables.

**2. User domain** - Add the user aggregate, the normalized email value object, the allowed-email rule, the typed user ID, the user repository and queries interfaces, and the user errors.

**3. Session domain** - Add the session aggregate with its sliding, absolute, and renewal-throttle rules, the token-hash value object, the session repository interface, and the authentication errors carrying the PRD messages.

**4. Users and sessions schema** - Define both tables in the Drizzle schema module and generate the migration so F01's runner applies it on boot.

**5. Persistence adapters** - Implement the user repository, user queries, and session repository over Drizzle, with mappers between rows and aggregates.

### Stage 2: Application Ports and Use Cases

**6. Authentication ports** - Define the Google OIDC gateway, PKCE, and session-token ports, and the YouTube and Telegram integration-status extension ports.

**7. Login use cases** - Implement starting a Google login and completing the callback: state and age validation, code exchange, the email rule, user upsert, session creation, and the typed rejection reasons that map to redirect error codes.

**8. Session and logout use cases** - Implement session authentication with lazy deletion of expired rows and throttled renewal, and logout by token hash.

**9. Current-user use case** - Implement the profile read that combines the user profile with both integration-status ports.

### Stage 3: Infrastructure Adapters

**10. Google OIDC gateway** - Implement the gateway with fetch for the token exchange and jose for ID-token verification against the configured JWKS and issuers, translating timeouts, non-success responses, and verification failures into the port's typed failures without logging tokens.

**11. Crypto adapters** - Implement PKCE generation with the S256 challenge, and session-token generation and hashing, on Node's crypto module.

**12. Default integration-status providers** - Implement the providers that report YouTube as disconnected and Telegram as not configured until F03 and F14 replace them.

### Stage 4: HTTP Interface and Wiring

**13. Route descriptor and server changes** - Add the public/required auth flag and the authenticated user to the route contract, register cookie support signed with the session secret, and attach the session check to every non-public route. Mark health and diagnostics routes public.

**14. Session middleware and cookie options** - Implement the session check hook, which reads the session cookie, delegates to the session use case, exposes the user to handlers, re-issues the cookie after renewal, and clears it on rejection. Keep both cookies' names and attributes in one module.

**15. Auth and profile routes** - Add the login start, callback, logout, and current-user handlers and routes with their Zod schemas so they appear in the OpenAPI document.

**16. Container wiring** - Register the new repositories, gateway, crypto adapters, default providers, use cases, handlers, and routes in the Awilix container.

### Stage 5: Environment and Mocks

**17. E2E topology and Google stubs** - Add the stub web service to the e2e compose and point the Google URLs at WireMock. Add WireMock mappings and response files for the token endpoint (allowed, other-email, unverified, failing, and expired-token codes) and for the JWKS endpoint, backed by pre-signed test ID tokens and a fixture generation script.

**18. Environment documentation** - Document the new variables, the Google redirect URI to register, and the override URLs in `.env.example`.
