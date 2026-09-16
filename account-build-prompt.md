# Build Prompt: Account Application (Retail Micro-Frontend Platform — Remote)

Use this as the instruction prompt for an AI coding agent (e.g. Claude Code) to scaffold and build the Account project.

---

## Context

You are building **Account**, one of three independently deployable remotes in a teaching example of enterprise Angular micro-frontend architecture. The other projects are the **Shell** (host, owned by a separate platform team), **Catalog**, and **Checkout** — you do not own their internals, only your contract with the Shell.

This project has two teaching goals, and they are not the same thing:

1. **Modern Angular (the "how"):** Angular 22, standalone components (no NgModules), Signals as the default reactivity model, zoneless change detection, `OnPush` as the default, modern control-flow syntax (`@if`/`@for`/`@switch`), lazy loading, and Native Federation as a **remote**.
2. **Enterprise architecture (the "why"):** owning a bounded context cleanly, consuming a shared runtime contract without reaching into the host's internals, being the most auth-sensitive remote on the platform without becoming a second source of truth for identity, independent deployability, and graceful behavior when the contract from Shell isn't available (e.g., session not yet resolved).

The code should teach both. **Comment generously and explain the architectural reasoning, not just what the code does.** A comment like `// gets user profile` is useless here. A comment like `// We read identity from the shared AuthSessionService contract exposed by Shell, not by managing our own token — Account is a consumer of session state, not its owner, so a token refresh implemented by Shell doesn't require an Account redeploy` is the target quality bar. Assume the reader is an experienced Angular developer who has *not* built a micro-frontend platform before.

---

## What Account Owns

Account is a **bounded context**: everything about the customer's relationship to their own profile and order history, and nothing else.

- **Profile screens**: view/edit basic profile information (name, email, preferences).
- **Order history**: list past orders and view order detail (use mock/stubbed order data — no real Checkout integration needed for this exercise).
- **Its own internal routing**: Account manages the routes *within* its own bounded context (e.g. `/account/profile`, `/account/orders`, `/account/orders/:id`); Shell only knows about the single mount point it delegates to Account.
- **Its own release cadence**: prove independence by structuring the project so it can be built, tested, and deployed with zero coordination from Catalog, Checkout, or Shell for a routine feature change.
- **Graceful degradation of its own screens** if data it depends on (profile API, order API — mocked here) is slow or fails, independent of whether Shell or other remotes are healthy.

## What Account Explicitly Does NOT Own

- Identity/session management itself. Account **consumes** the identity contract exposed by Shell (`AuthSessionService` or equivalent) — it does not implement its own login, does not manage its own token, and does not assume it's always populated (render a defined "session loading" or "not authenticated" state rather than assuming a user object exists).
- Global navigation, header, or layout chrome — that belongs to Shell.
- Cart, checkout, or payment logic — that belongs to Checkout. Order history here is read-only historical data, not live cart state.
- The Native Federation host configuration — Account only defines what it *exposes*, not how or when Shell chooses to load it.

---

## Requirements

### Angular / technical
- Angular 22, standalone components only, no NgModules.
- Zoneless change detection (no Zone.js dependency).
- `OnPush` as the default change detection strategy throughout.
- Signals for all local state — no bespoke RxJS state management unless justified in a comment for why signals weren't a fit.
- Modern control-flow template syntax (`@if`, `@for`, `@switch`) — no `*ngIf`/`*ngFor`.
- Native Federation configured as a **remote**: expose Account's root routed component (or a small set of exposed entry points) in the federation manifest so Shell can lazy-load it without needing Account's internal file structure.
- Use `resource()`/`httpResource()` for profile/order data fetching instead of manual subscriptions.

### Architecture
- **Bounded context clarity**: the folder structure should make it obvious what is internal to Account (profile/orders feature code) vs. what is the exposed remote entry point vs. what is consumed from the shared contract library.
- **Consuming the runtime contract, not the host**: Account depends only on the versioned shared library (design tokens/base components, `AuthSessionService`) that Shell also depends on — never on Shell's own internal code, services, or state. This is what lets Account and Shell be built and deployed independently.
- **Session-state resilience**: because Account is identity-heavy, explicitly handle three states wherever session matters — session resolved, session still loading, and unauthenticated — with a defined UI/component for each rather than assuming the happy path.
- **Isolation at the data layer**: if the (mocked) profile or order API is slow or errors, Account's own screens should degrade gracefully (loading state, retry, error state) without needing Shell or any other remote to know or care.
- **Independent deployability**: prove that Account can be rebuilt and redeployed on its own, and that Shell picks up the new version at runtime without a Shell rebuild — this should be evident from how the federation manifest/entry point is structured, not baked into Shell's build.
- **Version compatibility**: document how Account pins/verifies the version of the shared contract library it was built against, and what should happen if Shell reports a version it doesn't expect (fail loudly in dev, degrade gracefully in prod).
- **Observability**: add lightweight logging for Account's own lifecycle events (remote mounted, profile/order data load success/failure, load timing) in a structured form a platform team could correlate with Shell's own telemetry (same event shape/fields as Shell's logging service — this is what makes the contract "observability," not just "logging").

### Documentation deliverables
Alongside the code, produce:
1. `ARCHITECTURE.md` — explains Account's bounded context, how it consumes (not owns) the shared contract, its session-state handling strategy, and its data-layer resilience approach, written for another engineer joining the project.
2. `CONTRACT.md` — what Account exposes to Shell (its remote entry point/exposed modules, expected mount path) and what it expects to receive from Shell's shared contract (exact services/tokens it consumes, and the versions it's compatible with).

---

## Output format

- Provide the full file tree for the Account project first, then the code, file by file, with inline comments per the standard above.
- After the code, provide `ARCHITECTURE.md` and `CONTRACT.md`.
- End with a short "how to run this locally, standalone and federated into Shell" section.

Do not silently skip the session-state handling or observability requirements to save time — they are the point of the exercise, not optional polish.
