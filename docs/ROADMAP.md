# Saree Store: build roadmap

Work top to bottom. One slice per Claude Code session. Tick a slice when its checks pass and it is committed.

## How to run a slice

Paste this, with the slice number filled in:

```
Implement slice <N> from docs/ROADMAP.md.
Read only CLAUDE.md, that slice, the SPEC sections it lists, and the
STANDARDS section for the side you are working on (§2 backend, §3 frontend).
First give me a plan in 10 bullets or fewer and wait for my OK.
Then implement it with tests, and run lint, typecheck and tests.
Then run the standards-reviewer subagent on your changes and fix every blocker
it reports. Repeat until its verdict is PASS.
Finish with: what changed, what you assumed, what is left, and the reviewer's
final report. Tick the slice in the roadmap only after a PASS.
```

The `standards-reviewer` subagent lives in `.claude/agents/standards-reviewer.md`. It is read-only and checks the diff against `docs/STANDARDS.md` and the spec.

Habits that save usage:
- Start each slice in a fresh session (`/clear`), so old context is not re-read.
- Commit after every slice. If a session goes wrong, reset to the last commit instead of arguing with it.
- Review the plan before approving. Fixing a plan is far cheaper than fixing code.
- Use the default model for most slices. Use the strongest model you have for slices 6.1 and 6.2, and for the review slices marked **(review)**.

---

## Phase 0: Foundations

- [ ] **0.1 Monorepo scaffold.** SPEC §4. pnpm workspace with `apps/storefront` (Next.js App Router, TypeScript, Tailwind v4), `apps/admin` (Vite, React, TypeScript, Tailwind v4, TanStack Router and Query), `packages/ui`, `packages/api-client`, `packages/config` (shared tsconfig, ESLint, Prettier, Tailwind tokens). Root scripts from CLAUDE.md.
  Also set up the frontend enforcement from STANDARDS §3.1 to §3.3, §3.17 and §4: strict tsconfig flags, ESLint rule sets with zero warnings, path aliases, module-boundary and no-barrel rules, banned native dialogs, Prettier, `.npmrc` with `ignore-scripts`, pinned pnpm, pre-commit hook, commitlint, Vitest coverage gate.
  *Done when:* both apps start, share one component from `packages/ui`, `pnpm lint && pnpm typecheck` pass, and a deliberate violation (an `any`, a cross-app import) fails lint.
- [ ] **0.2 Backend scaffold.** SPEC §4, §7.1, §13. Spring Boot with web, validation, JPA, security, Flyway, actuator, OpenAPI. Package layout from CLAUDE.md. Global Problem Details error handler with `code`. Request id logging. `docker-compose.yml` with Postgres and MinIO. Profiles `local`, `test`, `prod`. Testcontainers base test class.
  Also set up the backend enforcement from STANDARDS §2.1, §2.2, §2.8 and §4: Spotless, static analysis with warnings as errors, ArchUnit layering tests, JaCoCo gate, injectable `Clock`.
  *Done when:* `./mvnw verify` passes, `/actuator/health` is up against the Compose database, and a deliberate violation (a controller using a repository) fails the build.
- [ ] **0.3 CI.** SPEC §12, STANDARDS §4. GitHub Actions for backend and frontends: lint, typecheck, test, build, coverage gates, dependency and secret scans. `.env.example` files. Protect `main`.
  *Done when:* CI is green on a pull request and red on a pull request that breaks a standard.

## Phase 1: Catalogue backend

- [ ] **1.1 Schema, part 1.** SPEC §6. Flyway migration for `admin_user`, `refresh_token`, `term`, `product`, `product_term`, `product_image`, `store_settings`, `audit_log`, with all indexes and constraints. Entities and repositories. Seed data for the `local` profile only.
  *Done when:* migration applies cleanly and repository tests pass.
- [ ] **1.2 Admin auth.** SPEC §7.3 (Auth), §11. Login, refresh with rotation and reuse detection, logout, me. Bootstrap admin. Role checks. Login rate limit.
  *Done when:* tests cover success, wrong password, expired token, rotated-token reuse, and role denial.
- [ ] **1.3 Taxonomy.** SPEC §6 (term), §7.2, §7.3. Admin CRUD and public `GET /taxonomy`.
- [ ] **1.4 Products, admin side.** SPEC §5.1, §5.2, §7.3 (Products, without mark-sold). Create, update with optimistic locking, publish rules, archive, code and slug generation, audit log.
  *Done when:* tests cover publish validation, immutability of code and slug, and version conflicts.
- [ ] **1.5 Products, public side.** SPEC §5.1, §5.2, §7.2. List with filters, sort, pagination, search; detail; similar; lookup; sitemap; `GET /store`. Computed price fields.
  *Done when:* tests prove drafts and archived products never leak, and sold-visibility rules hold.
- [ ] **1.6 API client generation.** SPEC §4. Export OpenAPI, generate TypeScript types and a typed fetch client into `packages/api-client`, add `pnpm gen:api`, and a CI check that fails when it is stale.

## Phase 2: Images

- [ ] **2.1 Image upload.** SPEC §7.3 (Images), §11. Presigned upload URL, confirm with server-side checks, alt text, primary, reorder, delete. MinIO locally, S3-compatible config for production.
  *Done when:* integration tests run against MinIO in Testcontainers and limits are enforced.

## Phase 3: Admin app

- [ ] **3.1 Shell and auth.** SPEC §9. Layout, navigation, login page, in-memory token, silent refresh, route guards by role, API client wiring, error and loading patterns.
- [ ] **3.2 Taxonomy screens.** SPEC §9.
- [ ] **3.3 Product list and form.** SPEC §9, §5.1, §5.2. Table with search and filters; create and edit form with React Hook Form and Zod; sale pricing; publish, unpublish, archive; server validation errors shown on fields.
- [ ] **3.4 Image manager.** SPEC §9, §7.3 (Images). Upload with progress, drag to reorder, set primary, alt text, delete.
- [ ] **3.5 (review) Admin review.** Ask Claude to review Phases 1 to 3 against the spec for security, consistency, and missing tests. Fix what it finds.

## Phase 4: Storefront

- [ ] **4.1 Shell and home.** SPEC §8.1, §8.2 (Mobile navigation). Layout, header, footer, sticky bottom navigation on phones, design tokens, home page sections, static pages as placeholders.
- [ ] **4.2 Listing.** SPEC §8.1, §8.2. Product cards, filters and sort in URL params, pagination, empty states. Mobile filter drawer.
- [ ] **4.3 Detail page.** SPEC §8.2. Gallery with zoom, price and badges, details, similar sarees, sold and reserved states.
- [ ] **4.4 SEO and cache refresh.** SPEC §8.3, §8.4. Metadata, Open Graph, JSON-LD, sitemap, robots, `/api/revalidate`, and the backend webhook fired after commit.
  *Done when:* a link-preview debugger shows image, title, and price, and editing a product in admin updates the storefront within seconds.

## Phase 5: Interest signals

- [ ] **5.1 Events.** SPEC §6 (event), §7.2, §10. Migration, `POST /events`, bot filter, rate limit, retention job. Storefront tracker utility with batching and visitor id. Track `PRODUCT_VIEW` and `LIST_VIEW`.
- [ ] **5.2 Wishlist.** SPEC §8.2. Persisted store, heart button, `/wishlist` page hydrated by lookup, events.
- [ ] **5.3 Share and WhatsApp.** SPEC §8.2. Share button, "I'm interested on WhatsApp", "Send my wishlist on WhatsApp", events. Unit tests for the message builder.

## Phase 6: Reservations

- [ ] **6.1 Reservation backend.** SPEC §5.3, §6 (reservation), §7.2, §7.3 (Reservations), §11. Migration, state machine service with product row locking, public and admin endpoints, anti-abuse rules, phone normalisation, audit log.
  *Done when:* every row of the §5.3 table has a test, invalid transitions return 409, and the concurrency test passes.
- [ ] **6.2 Expiry and cascades.** SPEC §5.3, §5.4. Scheduled expiry job, cancel pending on sold, `mark-sold` and `undo-sold` endpoints.
  *Done when:* tests use a controllable clock to prove expiry and idempotency.
- [ ] **6.3 Storefront reservation flow.** SPEC §8.2. Reserve form, success screen with reference and WhatsApp button, `/reservations` page, shopper cancel, error handling for unavailable products.
- [ ] **6.4 Admin reservations.** SPEC §9. Inbox with tabs, waitlist count, confirm, cancel, complete, extend, notes, manual reservation, new-request badge.
- [ ] **6.5 (review) Reservation review.** Ask Claude to try to break the reservation rules: races, double submits, expired holds, abuse. Fix what it finds.

## Phase 7: In-store and analytics

- [ ] **7.1 In-store view.** SPEC §9. Available-now grid, search by code, one-tap mark sold and reserve, phone-friendly.
- [ ] **7.2 Analytics backend.** SPEC §7.3 (Analytics), §10. Overview, products, sales by channel, inventory, insights.
- [ ] **7.3 Dashboard and analytics screens.** SPEC §9.
- [ ] **7.4 Settings and users.** SPEC §7.3, §9. Store settings and admin user management, OWNER only.

## Phase 8: Hardening and launch

- [ ] **8.1 Security, accessibility, performance pass.** SPEC §11, §12. Headers, CORS, rate limits, upload checks; keyboard and contrast checks; Lighthouse on mobile.
- [ ] **8.2 End-to-end tests.** SPEC §12. The four Playwright flows.
- [ ] **8.3 Deployment.** SPEC §13, §14. Dockerfile, hosting choice, staging and production environments, domains, backups with a restore test, error tracking, uptime checks.
- [ ] **8.4 Content and launch check.** Real About, Contact, and Privacy text; real products and photos; WhatsApp link previews tested on real phones; remove bootstrap admin variables.

---

## After v1

See SPEC §15. Likely first: new-reservation alerts, home page banners, bulk product import. Later: payments and orders, coupons, customer accounts, stock quantities, reviews, mobile app, WhatsApp automation, languages.
