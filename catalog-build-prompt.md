# Build Prompt: Catalog Application (Retail Micro-Frontend Platform — Remote)

Use this as the instruction prompt for an AI coding agent  to scaffold and build the Catalog project.

---

## Context

You are building **Catalog**, one of three independently deployable remotes in a teaching example of enterprise Angular micro-frontend architecture. The other projects are the **Shell** (host, owned by a separate platform team), **Checkout**, and **Account** — you do not own their internals, only your contract with the Shell.

This project has two teaching goals, and they are not the same thing:

1. **Modern Angular (the "how"):** Angular 22, standalone components (no NgModules), Signals as the default reactivity model, zoneless change detection, `OnPush` as the default, modern control-flow syntax (`@if`/`@for`/`@switch`), lazy loading, and Native Federation as a **remote**.
2. **Enterprise architecture (the "why"):** owning a bounded context cleanly, consuming a shared runtime contract without reaching into the host's internals, treating an external/legacy-shaped data source as an untrusted boundary you adapt at the edge, independent deployability, and graceful behavior under data-layer failure — Catalog is the highest-traffic, most latency-sensitive remote on the platform, so performance and resilience decisions matter here more than anywhere else.

The code should teach both. **Comment generously and explain the architectural reasoning, not just what the code does.** A comment like `// fetch products` is useless here. A comment like `// We map the Northwind "Products" shape to our own ProductSummary view-model at the edge, in one place — if the data source changes later, only this adapter changes, not every component that renders a product` is the target quality bar. Assume the reader is an experienced Angular developer who has *not* built a micro-frontend platform before.

---

## Data source: Northwind Traders on Supabase

Catalog's backing data is a **Supabase** project (hosted Postgres, free tier: 500 MB DB) seeded with the classic **Northwind Traders** sample schema, loaded via the community Postgres port (`pthom/northwind_psql` on GitHub — run its SQL script directly against the Supabase project's Postgres instance via the SQL editor or `psql`). Supabase auto-generates a REST API (PostgREST) and a GraphQL API over that schema with no backend code required — use the REST API for this exercise.

- **Setup assumption**: a Supabase project exists with the Northwind schema loaded (`products`, `categories`, `suppliers`, and optionally `orders`/`order_details` for reference). Catalog talks to it via PostgREST's auto-generated REST endpoints (e.g. `GET /rest/v1/products?select=*`), authenticated with Supabase's anon public API key for read access.
- **Row Level Security (RLS)**: Postgres tables on Supabase are inaccessible via the API by default until RLS policies are added. Part of this build is defining and documenting a read-only RLS policy on `products` and `categories` (public `SELECT`, no `INSERT`/`UPDATE`/`DELETE` from the client) — call this out explicitly in `ARCHITECTURE.md`, since "the API just doesn't return data" is a common and confusing failure mode until this is understood.
- **Do not use Supabase Auth for session/identity in Catalog.** Supabase happens to ship built-in Auth, but in this platform, Shell owns identity and exposes it via the shared `AuthSessionService` contract — that's the whole point of the platform's runtime contract. Wiring Catalog directly to Supabase Auth would quietly create a second, competing source of identity truth and defeat the architecture lesson. Catalog only ever talks to Supabase for **product data**, never for auth.
- Model Catalog's domain around **Products** (name, price, unit info, stock level, category, supplier) and **Categories** for browsing/filtering. Orders/Suppliers can be included as read-only supporting data if useful, but Catalog's job is browsing and search, not order management.
- Explicitly do **not** let the Northwind/Postgres schema leak into Catalog's component layer. Even ported to Postgres, Northwind's field and table shapes (e.g. `unit_price`, `units_in_stock`, `category_id`, PostgREST's embedded-resource response shape for joins) are a legacy-shaped external contract — adapt them at a data-access boundary into a clean, Catalog-owned view-model (`Product`, `Category`). This is itself part of the architecture lesson: an external/legacy data source is a boundary you defend, the same way Shell's contract is a boundary Catalog respects.

---

## What Catalog Owns

Catalog is a **bounded context**: everything about browsing, searching, and discovering products, and nothing about buying them.

- **Product browsing**: a paginated/filterable product list, sourced from the adapted Northwind product data.
- **Category navigation**: browse by category.
- **Product detail**: a single product view with the data needed to decide to buy (not to actually buy).
- **Search**: basic search over product name/description.
- **Its own internal routing**: Catalog manages routes within its own bounded context (e.g. `/catalog`, `/catalog/category/:id`, `/catalog/product/:id`); Shell only knows about the single mount point it delegates to Catalog.
- **Its own release cadence**: prove independence by structuring the project so it can be built, tested, and deployed with zero coordination from Checkout, Account, or Shell for a routine feature change.
- **Its own performance budget**: Catalog is the remote most customers hit first and most often — bundle size and load time here are architectural concerns, not afterthoughts.

## What Catalog Explicitly Does NOT Own

- Cart or checkout logic. A "buy" or "add to cart" action on a product detail page should be a clearly stubbed interaction point (e.g., emits an event / calls a shared contract method) rather than Catalog reaching into Checkout's territory to implement cart behavior itself.
- Identity/session management. If Catalog needs to know whether a user is signed in (e.g., to show personalized results later), it consumes the same `AuthSessionService` contract from the shared library that every remote consumes — it does not implement its own.
- Global navigation, header, or layout chrome — that belongs to Shell.
- The Native Federation host configuration — Catalog only defines what it *exposes*, not how or when Shell chooses to load it.

---

## Requirements

### Angular / technical
- Angular 22, standalone components only, no NgModules.
- Zoneless change detection (no Zone.js dependency).
- `OnPush` as the default change detection strategy throughout.
- Signals for all local state — no bespoke RxJS state management unless justified in a comment for why signals weren't a fit.
- Modern control-flow template syntax (`@if`, `@for`, `@switch`) — no `*ngIf`/`*ngFor`.
- Native Federation configured as a **remote**: expose Catalog's root routed component (or a small set of exposed entry points) in the federation manifest so Shell can lazy-load it without needing Catalog's internal file structure.
- Use `httpResource()` for all Supabase-backed data fetching (product list, category list, product detail, search via PostgREST's `?field=ilike.*term*` filter syntax) instead of manual subscriptions.
- Read the Supabase project URL and anon API key from environment configuration (e.g. Angular's `environment.ts` or a runtime-injected config, not hardcoded) — this is itself a small but real lesson: a remote's config, like its code, should be deployable independently of the other remotes.

### Architecture
- **Bounded context clarity**: the folder structure should make it obvious what is internal to Catalog (browsing/search feature code) vs. the exposed remote entry point vs. what is consumed from the shared contract library.
- **Anti-corruption layer for Northwind/Supabase**: a dedicated data-access module that adapts raw PostgREST response shapes into Catalog-owned models (`Product`, `Category`) used everywhere else in the app. No component or service outside this layer should know Postgres/PostgREST field names, snake_case conventions, or embedded-resource join shapes.
- **Consuming the runtime contract, not the host**: Catalog depends only on the versioned shared library (design tokens/base components, `AuthSessionService`) that Shell also depends on — never on Shell's own internal code, services, or state.
- **Cross-remote handoff point**: define, at the contract level, how a "buy this product" action on a Catalog page hands off to Checkout (e.g., a typed event on the shared event bus, or Checkout's exposed route) — implement the Catalog side of this handoff, stub the receiving end.
- **Resilience under data-layer failure**: if Supabase is slow, unreachable, rate-limited (free tier), or an RLS policy misconfiguration silently returns an empty result set, Catalog's screens should degrade gracefully (loading state, retry, empty/error state) without needing Shell or any other remote to know or care. This is especially important here since Catalog depends on a third-party hosted data source rather than infrastructure it fully controls.
- **Independent deployability**: prove that Catalog can be rebuilt and redeployed on its own, and that Shell picks up the new version at runtime without a Shell rebuild.
- **Version compatibility**: document how Catalog pins/verifies the version of the shared contract library it was built against.
- **Observability**: add lightweight structured logging for Catalog's own lifecycle events (remote mounted, product/category/search load success/failure, load timing, Northwind adapter errors), using the same event shape/fields as Shell's and Account's logging so a platform team can correlate across remotes.

### Documentation deliverables
Alongside the code, produce:
1. `ARCHITECTURE.md` — explains Catalog's bounded context, the Northwind/Supabase anti-corruption layer and why it exists, the RLS setup and why the API returns nothing without it, why Catalog deliberately does not use Supabase Auth, the buy-handoff contract with Checkout, and Catalog's resilience strategy — written for another engineer joining the project.
2. `CONTRACT.md` — what Catalog exposes to Shell (its remote entry point/exposed modules, expected mount path), what it expects from the shared contract library, and the shape of the "product selected for purchase" handoff event it emits for Checkout to eventually consume.

---

## Output format

- Provide the full file tree for the Catalog project first, then the code, file by file, with inline comments per the standard above.
- After the code, provide `ARCHITECTURE.md` and `CONTRACT.md`.
- Include clear setup instructions for the Supabase side as part of the deliverable: creating the project, loading the `pthom/northwind_psql` schema/data via the SQL editor, and the exact RLS policy SQL to enable public read on `products` and `categories`, since Catalog depends on this being in place to run.
- End with a short "how to run this locally, standalone and federated into Shell" section, including where to put the Supabase URL/anon key.

Do not silently skip the anti-corruption layer, the RLS setup, or the resilience requirements to save time — they are the point of the exercise, not optional polish.
