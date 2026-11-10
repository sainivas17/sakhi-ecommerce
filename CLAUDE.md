# Saree Store: guide for Claude Code

Catalogue and reservation site for a saree business. Shoppers browse, wishlist, share, say "I'm interested" on WhatsApp, and request a reservation. Admins manage products, reservations, and see analytics. **No payments, delivery, or customer accounts in v1.**

- Full spec: `docs/SPEC.md`
- Build order: `docs/ROADMAP.md`
- Coding standards (mandatory): `docs/STANDARDS.md`

Read only the spec sections a slice names. Do not load the whole spec unless asked.
Before writing code, read the STANDARDS section for the side you are working on (§2 backend, §3 frontend). The convention lists below are a summary; STANDARDS.md wins on any conflict.

## Repo layout

```
apps/storefront     Next.js (App Router), public site
apps/admin          Vite + React SPA, admin portal
packages/ui         Shared React components (no framework imports)
packages/api-client Types and client generated from the backend OpenAPI spec
packages/config     Shared tsconfig, ESLint, Prettier, Tailwind tokens
backend             Spring Boot REST API (Maven), package-by-feature
docs                SPEC.md, ROADMAP.md, STANDARDS.md
docker-compose.yml  Postgres + MinIO for local dev
```

## Commands (created in Phase 0; keep this list current)

```
docker compose up -d            # Postgres + MinIO
cd backend && ./mvnw spring-boot:run
cd backend && ./mvnw verify     # backend tests
pnpm dev:storefront
pnpm dev:admin
pnpm lint && pnpm typecheck && pnpm test
pnpm gen:api                    # regenerate packages/api-client from OpenAPI
```

## Working rules

1. One slice per session. Plan in 10 bullets or fewer and wait for approval before writing code.
2. The spec is the source of truth. If something is ambiguous or you think the spec is wrong, stop and ask. Do not silently deviate.
3. Use the current stable version of each tool, and check its official docs for APIs. Next.js and Spring Boot both changed defaults across recent major versions, so do not rely on memory.
4. Every slice ships with tests. Run lint, typecheck, and tests before saying it is done.
5. Finish each slice with: what changed, what you assumed, what is left. Tick the slice in `docs/ROADMAP.md`.
6. Never commit secrets. Config comes from environment variables. Update `.env.example` when adding one.
7. Conventional Commits (`feat:`, `fix:`, `chore:`, `test:`, `docs:`). Small commits.
8. Do not add a dependency without saying why.
9. Never disable a lint rule, skip a test, lower a threshold, or add a suppression comment to make a check pass. Ask instead.

## Backend conventions

- Package-by-feature under `com.sareestore`: `product`, `taxonomy`, `media`, `reservation`, `analytics`, `auth`, `settings`, `audit`, `common`.
- Thin controllers, transactional services, Spring Data repositories. DTOs are Java records. Never return JPA entities from controllers.
- Flyway for schema. Never edit an applied migration; add a new one.
- Errors use RFC 9457 Problem Details with a stable `code` field (see spec §7.1).
- Time is `Instant` in UTC (`timestamptz`). IDs are UUIDs. Money is a `long` in paise.
- Validate all input with Jakarta Validation. Mask phone numbers in logs.
- Reservation and availability changes go through the state machine service only (spec §5), inside a transaction that locks the product row.
- Tests: unit tests for services, integration tests with Testcontainers Postgres for repositories and controllers.

## Frontend conventions

- TypeScript strict, no `any`. API types come from `packages/api-client`; never handwrite them.
- Tailwind CSS v4 with tokens from `packages/config`. Mobile-first.
- Storefront: Server Components by default. Client components only for interaction (wishlist, filters, reserve form). Filters live in URL search params.
- Admin: TanStack Router (file-based, lazy route pairs), TanStack Query (`staleTime: 0` default), React Hook Form + Zod, Zustand for the auth token and global UI state.
- No barrel files, no feature-to-feature imports, path aliases only. Components live in `Name/Name.tsx`.
- Every on-screen query has a loading state. No layout shift, no empty state while loading.
- Format money with one shared helper (paise to `₹12,500`, `en-IN`).
- Accessibility: semantic HTML, visible focus, labelled inputs, alt text on every product image.
- Tests: Vitest + React Testing Library for logic and components, Playwright for the end-to-end flows in spec §12.
