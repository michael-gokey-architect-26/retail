# Build Prompt: Shell Application (Retail Micro-Frontend Platform)

Use this as the instruction prompt for an AI coding agent (e.g. Claude Code) to scaffold and build the Shell project.

---

## Context

You are building the **Shell** application for a teaching example of enterprise Angular micro-frontend architecture. This is one of four independently deployable projects: **Shell** (this one), **Catalog**, **Checkout**, and **Account**. The other three are separate remotes built and owned by other teams — you do not own their internals, only your contract with them.

This project has two teaching goals, and they are not the same thing:

1. **Modern Angular (the "how"):** Angular 22, standalone components (no NgModules), Signals as the default reactivity model, zoneless change detection, `OnPush` as the default, modern control-flow syntax (`@if`/`@for`/`@switch`), lazy loading, and Native Federation as the host.
2. **Enterprise architecture (the "why"):** bounded contexts, clear ownership boundaries, runtime contracts between independently deployed apps, isolation and blast-radius containment, independent deployability, shared-dependency management, failure handling, and observability — the layer of decisions that make four separately deployed apps feel like one coherent product.

The code should teach both. **Comment generously and explain the architectural reasoning, not just what the code does.** A comment like `// increments count` is useless here. A comment like `// We read cart count from a signal exposed by Checkout's contract, not by importing Checkout's internals — this keeps Shell decoupled from Checkout's implementation and lets Checkout redeploy without touching Shell` is the target quality bar. Assume the reader is an experienced Angular developer who has *not* built a micro-frontend platform before.

---

## What the Shell Owns

The Shell is the **composition layer**, not a feature app. It owns:

- **App shell / layout**: header, global navigation, footer — the chrome that stays constant across all remotes.
- **Auth session management**: sign-in state, session token, and the mechanism by which that token/identity is made available to remotes.
- **Native Federation host configuration**: the manifest/import-map that declares where each remote (Catalog, Checkout, Account) is loaded from, and the routing that lazily mounts each remote at its path.
- **The shared runtime contract**: the documented, versioned surface that remotes are allowed to depend on — shared design tokens/component library, an auth/session service, and a minimal cross-remote state mechanism (e.g., cart item count shown in the shell nav, sourced from Checkout without Shell knowing Checkout's internals).
- **Failure isolation**: if a remote fails to load or throws at runtime, the Shell must not go down with it. Define and implement a per-remote error boundary and fallback UI.
- **Observability**: structured logging and basic telemetry for remote load success/failure, load timing, and version mismatches — the Shell is the natural place to observe the health of the whole composed platform, since it's the only piece that sees all of them.

## What the Shell Explicitly Does NOT Own

- Catalog, Checkout, and Account business logic. Do not build real product listing, payment, or profile features here.
- For this exercise, stub the three remotes as minimal standalone Angular apps (their own separate projects) that expose one lazy-loadable route each, just enough to prove the federation, routing, and contract mechanisms work end-to-end. Keep them intentionally thin — the teaching focus of this build is the Shell's architecture, not their features.

---

## Requirements

### Angular / technical
- Angular 22, standalone components only, no NgModules.
- Zoneless change detection (no Zone.js dependency).
- `OnPush` as the default change detection strategy throughout.
- Signals for all local and shared state — no bespoke RxJS state management unless justified in a comment for why signals weren't a fit.
- Modern control-flow template syntax (`@if`, `@for`, `@switch`) — no `*ngIf`/`*ngFor`.
- Native Federation configured as the **host**. Each remote is declared in the federation manifest and loaded lazily via the Angular router (`loadComponent`/`loadChildren` pointing at the remote entry).
- Use `resource()`/`httpResource()` where the Shell needs async data (e.g., session/user info) instead of manual subscriptions.

### Architecture
- **Bounded contexts**: the repo/folder structure should make it visually obvious what belongs to the Shell's own domain (layout, auth, composition) vs. what is a contract surface (shared library) vs. what is a remote stub.
- **Runtime contract**: create a documented, versioned shared library (e.g. `shared-contract`) exporting: design tokens/base components, an `AuthSessionService` (signal-based), and a minimal cross-remote messaging mechanism (favor a typed event bus or shared injectable signal service over ad hoc globals — pick one, and comment on why).
- **Isolation**: wrap each federated remote's route in an error boundary component. If a remote fails to load (network failure, version mismatch, runtime exception), render a defined fallback UI and log the failure — never let one remote's failure blank the whole shell.
- **Independent deployability**: the Shell must be able to pick up a new deployed version of any remote without itself being rebuilt or redeployed. Prove this in how the federation manifest is structured (e.g., remote URLs resolved at runtime, not baked in at build time).
- **Shared dependency policy**: document (in comments and in the architecture doc below) how Angular version and shared-library versions are kept compatible across Shell and remotes, and what happens on a version mismatch.
- **Observability**: add a lightweight logging/telemetry service in the Shell that captures remote load attempts, load duration, and failures, with enough structure (remote name, version, timestamp, outcome) to be useful in a real incident.

### Documentation deliverables
Alongside the code, produce:
1. `ARCHITECTURE.md` — explains the bounded contexts, the runtime contract, the failure-handling strategy, and the shared-dependency policy, written for another engineer joining the project.
2. `CONTRACT.md` — the versioned surface Shell exposes to remotes and expects from them (what a remote must implement/expose to be mountable by this Shell).

---

## Output format

- Provide the full file tree for the Shell project first, then the code, file by file, with inline comments per the standard above.
- After the code, provide `ARCHITECTURE.md` and `CONTRACT.md`.
- End with a short "how to run this locally" section (assuming the three remotes are stubbed as described).

Do not silently skip the failure-handling or observability requirements to save time — they are the point of the exercise, not optional polish.
