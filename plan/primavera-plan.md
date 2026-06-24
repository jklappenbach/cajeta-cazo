# primavera roadmap

primavera is Cajeta's opinionated enterprise framework — the **policy** layer over
the language's neutral **mechanism** (`@Inject`, `FiberLocal`, reflection). This
plan tracks the increments from today's request-scope foundation to a full
web/service framework with first-class testability.

## Design principle: mechanism in the language, policy in primavera

The language ships primitives that don't presume a framework:

- `@Inject` — the one DI primitive that stays in the language.
- `FiberLocal` / `FiberContext` — ambient per-request state that propagates
  across both fan-out and handoff.
- reflection — construct + invoke by metadata.

primavera composes those into an opinion about enterprise services. Anyone who
dislikes the opinion keeps the primitives and builds their own framework — the
core never forces primavera. (`@Component` and the AOP annotations are *recognized by
the compiler today* as an interim; their spec/docs now live here in
`docs/AspectModel.md`, and the long-term aim is to re-home their
implementation behind a language-level annotation-processing extension point so
the core carries no framework policy. See the cajeta-two side for that work.)

## Phase 1 — Request scope ✅ (shipped v0.1.0)

- `org.cajeta.primavera.context.ScopeMap` — owning, name-keyed per-scope bean store.
- `org.cajeta.primavera.context.RequestScope` — request scope over `FiberLocal`,
  correct under both fiber-per-request and completion-port executors.
- Runtime self-test (`selftest/SelfTest`) — enter / store / lookup / teardown.

## Phase 2 — Session scope

A session outlives a single request and is shared across a client's requests, so
it is **not** a `FiberLocal` binding. Build:

- `SessionStore` — owning, id-keyed store of session `ScopeMap`s; create / get /
  invalidate; **expiry** (idle + absolute TTL); **concurrency-safe** access
  (the store is shared across all in-flight requests — guard with the stdlib
  `Lock` / `LockGuard`; timestamps from `Clock.millisTime()`).
- `SessionScope` — ambient access to the *current* session for a request:
  bind the session **id** (cheap, owned `String`) via `FiberLocal` for the
  request extent, resolve id → store on access. The session map's lifetime is
  the store's, never the request binding's.

> **BLOCKER (found 2026-06-16): cajeta collections are non-owning.**
> A correct *evicting* session store needs to hold session bags it OWNS (so an
> evicted/invalidated session — and its `ScopeMap` of beans — actually frees)
> AND remove them on expiry/invalidate. Verified empirically (`--emit=exe`
> drop-tracking):
> - `ArrayList<T>` does **NOT drop its elements** when it drops — they LEAK.
>   (The destructor mechanism works for plain locals; the list just never drives
>   element drops.) It also has no `remove`/`clear`.
> - Reassignment doesn't drop the overwritten value; there's no user-level
>   explicit-drop keyword.
> - `HashMap<K,V>` has `remove` but holds **borrowed** values
>   (`vals[idx]=value`), so eviction would leak the bag graph.
>
> Consequence: even the request bag (`ScopeMap`) currently **leaks** its beans at
> request end. A real owning collection needs **compiler-level** support: the
> synthesized drop for a container must drop live elements `[0, sizeCount)` when
> `T` is an owned reference type (the compiler knows T's ownership; user template
> code can't branch on it, and `__cajeta_class_virtual_drop` would crash on a
> primitive `T`). This is a **language memory-model decision** — escalated to the
> language author. Tracked in project memory `collections-dont-own-elements`.
> Phase 2 (and leak-free request scope) is blocked on it.

## Phase 3 — Component lifecycle integration

- Wire request/session scope to `@Component` allocation modes so a request- or
  session-scoped component resolves through `RequestScope` / `SessionScope`
  get-or-create instead of the singleton path.
- Reconcile with `ALLOCATE_*` modes documented in `docs/AspectModel.md`
  (`CALL_SCOPE` is currently stubbed in the compiler).

## Phase 4 — Web request/session model + pluggable executor

- HTTP request/response model; handler API.
- **Pluggable executor** behind one handler API:
  - fiber-per-request (simple, scales on Cajeta fibers);
  - threadpool-over-completion-ports (async I/O, max throughput).
- Scope propagation correct under both (Phase 1's FiberLocal foundation already
  provides the carry-across-handoff; the executor wires it at the boundary).

## Phase 5 — primavera-side unit-test helpers (on cajeta-unit)

The test capability for primavera's features lives **in primavera**, built on the neutral
[cajeta-unit](https://github.com/jklappenbach/cajeta-unit) framework but knowing
primavera's internals:

- mock `@Request` / `@Session`-scoped beans; seed request/session context.
- run a handler under a **chosen executor** (thread-per-request vs
  completion-port) so a test can assert behaviour under each.
- assert request/session **isolation** (no cross-request leakage) and that scope
  **follows the request across a handoff** — the bug a thread-per-request-only
  unit test can't see. A harness that simulates the thread-hopping
  completion-port path is what catches "someone used thread-local and it leaks
  under load."
- override the `@Inject` DI context with mocks/fakes for testing.

## Cross-cutting cleanups

- **Remove the static-field-method-receiver workaround** in `RequestScope` once
  the cajeta-two codegen bug is fixed (see project memory
  `static-field-method-receiver-segv`).
- **Once-guard** the lazy `FiberLocal` key init against a first-touch race.
- **Wire the real dependencies** (`org.cajeta.logging` runtime, `org.cajeta.unit`
  dev) in `cajeta.json` once they are resolvable (registry publish or
  path-dependency); primavera's web/component layer logs through cajeta-logging.

## Dependency on cajeta-two work

- A linker / build-system **dead-code-elimination** pass so a service links only
  the primavera classes it uses (embedded/robotics targets must fit an EPROM). Today
  unused classes are pinned by the reflection registry; this is tracked on the
  cajeta-two side, and primavera's per-feature modularity assumes it lands.
