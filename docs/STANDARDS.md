# Saree Store: coding standards

These rules are mandatory. Most are enforced by tooling, so a violation fails the build, not just a review.

**Policy for Claude Code:** never disable a lint rule, skip a test, lower a threshold, or add a suppression comment to make a check pass. If a rule seems wrong for a case, stop and ask.

## 1. Rules for the whole repo

- **Definition of done:** code, tests, lint, typecheck, and build all pass; no new warnings; docs and `.env.example` updated; roadmap slice ticked.
- **Git:** work on a branch (`feat/1.4-products-admin`), never push to `main`. Conventional Commits, checked by commitlint. One slice per pull request, kept small.
- **No dead weight:** no commented-out code, no unused exports, no `TODO` without an issue reference.
- **Secrets:** never in code, tests, logs, or commits. A secret scanner runs in CI.
- **Dependencies:** each new one needs a stated reason. Versions pinned by lockfile. Vulnerability scan in CI.
- **Naming:** names say what a thing is or does. No abbreviations except common ones (`id`, `url`, `dto`).
- **Personal data:** customer name and phone never appear in logs, URLs, error messages, or analytics.

## 2. Backend (Java, Spring Boot)

### 2.1 Structure

- Package-by-feature. Inside a feature: `api` (controllers, DTOs), `domain` (entities, services, rules), `persistence` (repositories).
- Dependency direction is controller → service → repository. A controller never touches a repository.
- A feature never uses another feature's repository or entity directly. It calls that feature's service.
- `common` holds only cross-cutting code (errors, pagination, clock, security helpers). It depends on no feature.
- These rules are written as ArchUnit tests and run with the test suite.

### 2.2 Code style

- Formatting by Spotless (google-java-format). The build fails on unformatted code.
- Constructor injection only. All injected fields are `final`. No field `@Autowired`.
- No Lombok. DTOs are records. Entities have explicit accessors and no public setters for state that has rules (use intent methods such as `markSold(...)`).
- Methods stay short and do one thing. No boolean flag parameters that switch behaviour; write two methods.
- Never return `null` collections. `Optional` is for return values only, never for fields or parameters.
- No magic numbers or strings. Use constants, enums, or typed configuration.
- Static analysis (Error Prone or Checkstyle) runs in the build with warnings as errors.

### 2.3 API layer

- Controllers only translate HTTP to service calls. No business logic, no transactions.
- Every request DTO is validated with Jakarta Validation. Every response is a DTO, never an entity.
- Correct status codes: 200, 201 with `Location`, 202, 204, 400, 401, 403, 404, 409, 429.
- Errors go through the single global handler as Problem Details with a stable `code`. No stack traces or SQL messages reach the client.
- Every list endpoint is paginated with a capped page size.
- `/api/v1` is never broken. Additive changes only; breaking changes need `/api/v2`.
- Every endpoint has OpenAPI annotations good enough to generate a usable client.

### 2.4 Services and transactions

- `@Transactional` on service methods only. Queries use `readOnly = true`.
- No network calls (object storage, webhooks, email) inside a database transaction. Publish an event and handle it after commit.
- Availability and reservation changes happen only in the state machine service. No other code sets those fields.
- Time comes from an injected `Clock`. Never call `Instant.now()` directly in domain code.
- Domain failures throw specific exceptions. Never catch `Exception` broadly, and never swallow an exception.

### 2.5 Persistence

- `spring.jpa.open-in-view=false` and `ddl-auto=validate`. Schema changes only through Flyway (`V<n>__<description>.sql`). Applied migrations are never edited.
- Associations are `LAZY`. No N+1 queries: use fetch joins, entity graphs, or projections. Integration tests assert query counts on list endpoints.
- No unbounded `findAll()`.
- Integrity lives in the database too: `NOT NULL`, foreign keys, unique and check constraints, not only in Java.
- Queries are parameterised. No string-concatenated SQL or JPQL.

### 2.6 Security

- Deny by default. Every admin endpoint declares its required role explicitly.
- Each admin endpoint has tests for 401 (no token) and 403 (wrong role).
- Input is validated at the boundary; output is encoded by the JSON serializer only.
- Configuration uses typed, validated `@ConfigurationProperties`. No scattered `@Value`.

### 2.7 Logging

- SLF4J with parameterised messages. Structured JSON in production, with a request id.
- `ERROR` for failures needing attention, `WARN` for handled problems, `INFO` for business events (reservation confirmed, product sold), `DEBUG` for diagnostics.
- Phone numbers are masked. Tokens and passwords are never logged.

### 2.8 Tests

- Unit tests for domain logic; integration tests with Testcontainers Postgres. No H2.
- Test names describe behaviour: `confirm_whenProductAlreadyReserved_returnsConflict`.
- Arrange, act, assert. One behaviour per test. No `Thread.sleep`; use the controllable clock.
- JaCoCo gate: at least 80% line coverage on `domain` packages. Every reservation transition, valid and invalid, has a test.

## 3. Frontend (TypeScript, React, Next.js, Vite)

Applies to both apps unless a rule names one. "Admin" means `apps/admin` (Vite, TanStack Router and Query). "Storefront" means `apps/storefront` (Next.js).

### 3.1 TypeScript

- `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`, `noFallthroughCasesInSwitch`, `noUnusedLocals`, `noUnusedParameters`, `forceConsistentCasingInFileNames` are on.
- No `any`; use `unknown` and narrow. No `@ts-ignore`. No non-null `!` assertions. No `as` on API or other external data.
- Index access returns `T | undefined`. Handle the `undefined` case.
- API types come only from `packages/api-client`.
- Anything from outside the type system is parsed with Zod before use: environment variables, `localStorage`, URL params, form input.
- `import type` for types (`consistent-type-imports`).
- `interface` for component props. Wrapper components extend `ComponentPropsWithoutRef<'element'>`.
- Discriminated unions over enums and optional-everything objects.
- Never mix `??` and `||` without parentheses.
- Public functions have explicit return types.

### 3.2 Lint, format, scripts

- ESLint flat config: `typescript-eslint` strict type-checked, `react-hooks`, `jsx-a11y`, import ordering with no cycles, `no-floating-promises`, and `no-restricted-syntax` banning `window.confirm`, `alert`, `prompt`. Run with `--max-warnings 0`.
- Prettier: single quotes, trailing commas, 100-character width, Tailwind class-sorting plugin.
- No `eslint-disable` without a one-line reason and approval.
- In chained scripts, custom checks run before ESLint, so a lint failure cannot hide them.

### 3.3 Structure and imports

Each app's `src/`:

```
components/ui        atomic primitives (app-specific; shared ones live in packages/ui)
components/layout    shell pieces
components/shared    cross-feature composites (tables, filter panels)
features/<name>      self-contained feature: components, hooks, queries, schemas
hooks  lib  services  stores  types  utils
routes (admin) or app (storefront)
```

- **No barrel files.** Import the file directly (`@components/ui/Button/Button`), never a folder `index.ts`.
- **Path aliases**, never deep relative paths: `@components`, `@features`, `@hooks`, `@lib`, `@services`, `@stores`, `@types`, `@utils`.
- A feature never imports from another feature. Code two features need moves to `components/shared`, `hooks`, or `utils`. Lint-enforced.
- Apps never import from other apps. `packages/ui` imports nothing from Next.js, TanStack Router, or any app.
- Shared code goes in `packages/*` only when both apps really use it.
- Route files are thin: params, search validation, guards, loaders, then hand off to a feature component.
- Tree-shakeable imports only (`import { debounce } from 'lodash-es'`).

### 3.4 Naming and files

- Components: `PascalCase`, one per file, in their own folder (`Button/Button.tsx`, `Button/Button.test.tsx`). Hooks: `useThing.ts`. Other files: `camelCase.ts`. Framework-named files (`page.tsx`, `layout.tsx`, route files) follow the framework.
- Named exports only, except where the framework requires a default export.
- Handlers in a component: `handleClick`. Handler props: `onClick`. Boolean props: `is` / `has` / `can` prefix.
- Constants: `UPPER_SNAKE_CASE`. No magic strings or numbers.
- Tests and small sub-components sit next to the component they belong to.

### 3.5 React patterns

- Function components only. The one exception is an error boundary class.
- **No `forwardRef`.** `ref` is a normal prop in React 19.
- `use(context)` over `useContext` in new code. Data still comes from TanStack Query or Server Components, not `use(promise)`.
- **Never derive state in an effect.** Compute during render, or `useMemo` if expensive.
- **Never fetch in an effect.**
- If an effect only reacts to a user action, move the logic into the event handler.
- **Never seed form or editor state from query data in an effect.** Use a lazy `useState` initializer, form `defaultValues`/`values`, or remount with `key={recordId}`.
- No mutation during render; side effects only in handlers or effects.
- Memoise only where it pays: a value passed to a `React.memo` child, an effect or memo dependency, or a truly expensive computation. `React.memo` on hot list rows and cards, not everywhere. Remove hand-memoisation if the React Compiler is enabled.
- Early returns over nesting.

### 3.6 Component design

- Extract when used 3 or more times, or when it is a stable unit (`Card`, `PageHeader`, `EmptyState`). Not for line count alone.
- Split a component into a hook plus sub-components when it passes about 200 lines, 5 `useState`, or 3 `useEffect`.
- Variant props (`variant="primary"`) over a pile of boolean props.
- Composition over configuration.
- No prop spreading into the DOM, except on the root of a primitive wrapper typed with `ComponentPropsWithoutRef`.
- No business logic in JSX.
- Lists have stable keys (never the index for dynamic lists).

### 3.7 State

| State | Tool |
|---|---|
| Shareable by link (filters, sort, page, tab, search) | URL search params, first choice |
| Server data | TanStack Query (admin); Server Components (storefront) |
| Global UI (auth token, toasts, sidebar) | Zustand (admin) |
| Wishlist, visitor id | Zustand with persist (storefront) |
| Component-scoped | `useState` / `useReducer` |
| Forms | React Hook Form |
| Derived | Computed inline |

- Never copy server data into Zustand or `useState`.
- **Zustand: always subscribe with a selector** (`useStore((s) => s.field)`). Never destructure the whole store. Multi-field selects use `useShallow`.
- Avoid React Context for shared state. If one is unavoidable, memoise its `value` and split contexts by update frequency.

### 3.8 Data fetching (admin)

```
Component (useQuery) → query factory (queryOptions) → service (typed promise, owns the URL) → API client
```

- All HTTP goes through the one client from `packages/api-client`. No raw `fetch` in components or query factories.
- Services own URLs: one file per API domain, flat functions. Use `URLSearchParams` and `encodeURIComponent`; never concatenate query strings.
- `import.meta.env` is read only in one config module. `VITE_*` values are never secrets.
- **Errors:** the client turns a Problem Details response into one typed `ApiError` with `code` and `errors[]`. Components branch on `code` and map `errors[]` onto form fields.
- **`staleTime: 0` is the default.** Reservations and stock are multi-user, so refetch on mount. `staleTime: Infinity` only for reference data (taxonomy, store settings), and say so when you use it.
- **Query keys are stable** and come from one key factory per feature. No inline object or array literals that change identity each render.
- **One owner per query.** If parent and child need the same data, the parent queries and passes it down.
- Retry once on network or 5xx errors. Never retry 4xx.
- No waterfalls: independent queries fire in parallel.
- `enabled` takes data dependencies (`!!id`) and open flags (`isOpen`) only. Never a role check; the backend decides.
- **Mutations** invalidate every affected query in `onSuccess`, including companion counts (the reservation summary badge, tab counts). `onError` always shows feedback through the shared toast helper. Never fail silently.
- A global `onError` on the QueryClient reports to the error tracker.

### 3.9 Loading states (both apps)

Every query whose result renders on screen needs a signalled loading state. Content that pops in after first paint and shifts the layout is a bug.

- Default: fold the query into the surface's loading gate (`primary.isLoading || secondary.isLoading`).
- For deferrable data: keep the container mounted and show a skeleton at its final size, so resolving swaps content, never geometry.
- Never `data && <Section />` for a post-paint query.
- Never show an empty state while loading. Empty renders only when `!isLoading && data.length === 0`.
- Loaders come from `packages/ui`, picked by scope (inline spinner, panel loader, page loader). Never hand-write a spinner or rotation keyframes.
- Storefront: every image reserves its space with explicit dimensions or aspect ratio.

### 3.10 Roles and 403s (admin)

- UI for an action the role cannot do is **hidden**, not disabled (`{isOwner && <Button />}`). Empty-state buttons use the same check as header buttons.
- Client-side role checks are for the user's convenience, not security. The backend authorises.
- One `isForbiddenError(error)` detector and one `<Forbidden>` component render a 403: inside the table's empty slot on a list, full page on a detail route.
- Route guards block rendering until auth state is known, so no page flashes to the wrong user.

### 3.11 Forms

- React Hook Form with a Zod schema as the single source of validation and types. `register()` for native inputs, `Controller` for custom ones.
- Flow: submit → `mutateAsync` → invalidate → reset or navigate. Server `errors[]` map onto the matching fields.
- **Never block silently.** Save is never disabled for a validation reason; it is disabled only while the mutation is in flight. On a failed submit, show the field errors, move focus to the first invalid field, and toast if that field is off screen.
- After `trigger()`, read errors with `getFieldState(name)`, not the `errors` render snapshot.
- Every input has a visible `<label htmlFor>`; errors link with `aria-describedby`.
- **Pickers:** a catalogue that can grow without bound (products in the manual reservation form) uses an async paginated select with backend search. Never bulk-fetch `size: N` into static options. Static selects are for small bounded lists (taxonomy terms, enums).

### 3.12 Tables, URL state, routing (admin)

- Table `page`, `size`, `sort`, filters, `tab`, and `search` are URL search params, validated with Zod in `validateSearch`. Optional params are optional in the type.
- Sort, size, filter, and search changes reset `page` to 0 and navigate with `replace: true`.
- Every table has backend-driven search.
- Status tabs are URL-persisted. Tab counts come from a counts query that is invalidated alongside the list.
- "Back to list" from a detail page restores the list's last search, filters, sort, and page.
- Deep-link params (`?tab=`): closing removes only that param, never `search: {}`.
- Row actions use the table's row-actions slot, not an extra column.
- Path verbs are only `new` and `$id`. Never `/add`, `/create`, `/view`, `/list`.
- Type-safe navigation with typed routes and `params`; no hand-built path strings.
- **Lazy routes:** every leaf route is a `.tsx` + `.lazy.tsx` pair. The eager half holds only `validateSearch`, `beforeLoad`, `loader`, and static data; the component lives in the lazy half. Check the entry-chunk size after `pnpm build`.
- A `$param` route can match same-depth static paths. When adding one, check its siblings.
- Heavy libraries (charts, drag and drop, image tools) load with dynamic `import()`.

### 3.13 Error boundaries, modals, dates

- Each top-level layout wraps its outlet in an error boundary reset by a `resetKey={pathname}` prop, not `key={pathname}`.
- Suspense is per feature, always paired with an error boundary, never at the app root.
- **Modals** use the shared `Modal` and confirm dialog from `packages/ui`. Never hand-roll an overlay. Footer: secondary then primary (danger when destructive), right-aligned. They trap focus, close on `Escape`, and restore focus.
- **Never `window.confirm`, `alert`, or `prompt`.** Lint forbids them.
- **Dates:** the API sends instants in UTC. Render them only through the shared date helper, in the viewer's local time. No bare `new Date(value)` formatting and no private `formatDate` per feature. If a date-only field is ever added, it is a calendar date and must be formatted from its own year, month, and day, not shifted by time zone.
- Display enum values through one shared `toTitleCase()` helper.

### 3.14 Next.js storefront

- Server Components by default. `"use client"` only on leaf components that need interaction.
- No secrets in `NEXT_PUBLIC_*`. Server-only modules are marked so client code cannot import them.
- All images use the framework image component with explicit sizes and alt text.
- Every page exports metadata. No client-side fetching for content that can render on the server.
- User-facing strings for a feature live in one constants module, so they can be translated later.

### 3.15 Styling and icons

- Tailwind utilities with tokens from `packages/config`. No raw hex colours, no arbitrary spacing values.
- No inline `style={{…}}`, except for truly dynamic values (position, progress width).
- Mobile-first: base styles target phones; larger screens add breakpoints.
- Spacing between fields comes from the container's `gap`, not margins on fields.
- Repeated class groups become a component, not a copied string.
- **Never inline an SVG in a component.** Each icon is its own file in `packages/ui/icons/<Name>Icon.tsx`, with `aria-hidden="true"` and `stroke="currentColor"` where it applies. Check the folder before adding one.

### 3.16 Accessibility and security

- Semantic elements (`button`, `a`, `nav`, `main`, `dialog`, `form`, `label`). No clickable `div`.
- Icon-only buttons have `aria-label`. Decorative images use `alt=""`; content images have descriptive alt text.
- Never convey state by colour alone. Text contrast at least 4.5:1. Focus always visible. Heading levels do not skip.
- Do not suppress `jsx-a11y` warnings without a written reason.
- No `dangerouslySetInnerHTML`. External links use `rel="noopener noreferrer"`.
- Object URLs made for upload previews are revoked on change and on unmount.
- **Logout** is one ordered sequence used everywhere: clear the query cache, clear the auth store, then hard-redirect to login.
- **Stale chunk after a deploy (admin):** on a chunk-load error, reload once, guarded by a `sessionStorage` flag so it cannot loop.

### 3.17 Dependencies and supply chain

- `.npmrc` has `ignore-scripts=true`. Only packages that truly need install scripts are allowlisted (`onlyBuiltDependencies`).
- The pnpm version is pinned exactly (`packageManager`, and in every Dockerfile). Never `@latest`.
- Lockfile always committed. CI and Docker use `pnpm install --frozen-lockfile`.
- Before adding a package: check it is maintained and tree-shakeable, and commit `package.json`, lockfile, and allowlist together. Remove unused dependencies.

### 3.18 Tests

- Vitest and React Testing Library. Mock the network with MSW, never by mocking `fetch`.
- Test behaviour, not implementation. Query by role or label; `data-testid` only as a last resort.
- Hooks via `renderHook` with a fresh `QueryClientProvider` per test.
- Cover validation, conditional rendering, loading, error, and empty states.
- Pure logic lives in `utils/` and is unit-tested apart from rendering.
- Coverage gate: at least 80% on helpers, stores, and hooks.
- Playwright covers the four flows in SPEC §12. A bundle-size check runs in CI.

### 3.19 Verification and docs

- **A green gate is not proof.** Typecheck, lint, and build all pass on code with dead redirects, broken layouts, and layout shift. For routing, guards, redirects, and anything visual, run it in a browser before calling it done.
- Report honestly: what was verified, what was not, and what still needs a manual walk.
- A feature with non-obvious decisions or gotchas records them in `features/<name>/FEATURE.md`.
- Code comments stay short (1 to 3 lines). Depth goes in `FEATURE.md`.

## 4. How the rules are enforced

| Stage | What runs |
|---|---|
| Editor | ESLint, Prettier, TypeScript; Java formatter |
| Pre-commit hook | Format and lint staged files; commitlint on the message |
| Backend build (`./mvnw verify`) | Spotless check, static analysis, unit and integration tests, ArchUnit rules, JaCoCo gate |
| Frontend (`pnpm lint && pnpm typecheck && pnpm test`) | ESLint with zero warnings, strict typecheck, Vitest with coverage gate |
| CI on every pull request | All of the above, builds, API client freshness check, dependency and secret scans, bundle-size check |
| Repository settings | `main` is protected: pull request and green CI required |

## 5. Frontend pre-merge checklist

- [ ] Type-safe: no `any`, no `as` on external data, everything external parsed at the boundary.
- [ ] No `useEffect` for fetching or derived state; no state seeded from query data in an effect.
- [ ] Zustand read through selectors only.
- [ ] No barrel files; no feature-to-feature imports; path aliases used.
- [ ] Query keys stable and from the key factory; no role checks in `enabled`.
- [ ] Every visible query has a loading state; no empty state while loading; no layout shift.
- [ ] Mutations invalidate affected queries (including counts) and show errors.
- [ ] Role-restricted UI is hidden, not disabled; 403s render through `<Forbidden>`.
- [ ] Forms never disable Save for validation; failed submits focus the first invalid field.
- [ ] Tables keep state in the URL, search on the backend, and use async selects for unbounded catalogues.
- [ ] Lazy route split is correct (no components in the eager half); no `$param` collisions.
- [ ] Tailwind tokens only; no raw hex; no repeated inline styles; no hand-written spinners.
- [ ] Icons in their own files; modals on the shared components; no native dialogs.
- [ ] Keyboard operable, labelled, semantic, not colour-only.
- [ ] Dates rendered through the shared helper.
- [ ] Lint clean without suppressions; behaviour checked in a browser, not only by the gate.
