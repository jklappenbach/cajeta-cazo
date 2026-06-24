# Request scope

primavera's request scope is the per-request bag of framework-managed beans. It is
the runtime that makes request-scoped `@Component` instances possible: created
once per request, reachable ambiently anywhere on the call path, and dropped
when the request ends.

## The one idea: key on the *logical request*

A request-scoped bean must be isolated per request and reachable without
threading a context parameter through every call. The naive implementation — a
thread-local — is correct *only* under thread-per-request. primavera keys on the
logical request instead, by building on `cajeta.concurrent.FiberLocal`. That
single choice makes one mechanism correct under both webserver concurrency
models:

| Model | How the request maps to execution | Why FiberLocal is correct |
|---|---|---|
| **thread / fiber-per-request** | one fiber owns the request end to end | the scope *is* that fiber's binding |
| **threadpool over completion ports** (epoll / io_uring / IOCP) | the request's continuation hops pool threads as I/O completes | the binding travels with the request: structured fan-out inherits it; an unstructured handoff carries a `FiberContext` snapshot. The scope follows the request across thread hops; a thread-local here would read another request's beans |

This is exactly why request scope belongs in a library over a *neutral*
primitive, not hard-wired to threads: the primitive already solved propagation.

## API

`org.cajeta.primavera.context.RequestScope` (all static):

| Method | Effect |
|---|---|
| `enter(() -> void body)` | run `body` with a fresh, empty request scope bound for its dynamic extent; the bag drops on return **or** throw |
| `isActive()` | whether a request scope is active on this fiber |
| `current()` | the active `ScopeMap`; throws if none active |
| `has(String typeName)` | whether a bean is bound for `typeName` (false, not throw, when no scope active) |
| `lookup(String typeName)` | the bean for `typeName`, or null |
| `store(String typeName, #Object instance)` | bind `instance` under `typeName`, taking ownership for the request |

Beans are keyed by fully-qualified type name (`String`). The typical pattern is
get-or-create:

```cajeta
if (RequestScope.has(key) == false) {
    RequestScope.store(key, #(heap Bean()));
}
Bean b = (Bean) RequestScope.lookup(key);
```

## Ownership & lifecycle

`ScopeMap` holds the beans in an `ArrayList<Object>`; a parallel key list is the
lookup index — a linear scan, the right structure for the handful of beans a
typical request touches. The `ScopeMap` itself is owned by the `FiberLocal`
binding created in `enter`, so when `enter`'s body returns or throws, the
binding pops and the `ScopeMap` is freed. The *scope* cannot outlive the
request.

> **KNOWN LIMITATION (v1): scoped beans leak.** `ArrayList<T>` does not drop its
> elements (verified — cajeta collections are non-owning today), so the beans in
> a `ScopeMap` are **not** freed at request end; their destructors do not run.
> Harmless for a short-lived process, wrong for a long-running server. The fix is
> a language-level owning-collection change (compiler-synthesized element drop),
> tracked in `plan/primavera-plan.md`. Don't rely on bean teardown at scope end yet.

## Implementation notes & v1 limitations

- **Lazy key init.** The process-wide `FiberLocal` key is created lazily
  through `ensure()` rather than a static-field initializer (object-typed static
  initializers aren't reliably run before first use in the current toolchain).
  The owned static `SCOPE` is never returned by value or bound to a local —
  doing so aliases ownership and the borrowing local would drop the singleton on
  scope exit; accessors dispatch methods on `RequestScope.SCOPE` directly. (An
  earlier codegen bug made *direct* static-field method dispatch fault; that is
  fixed upstream in cajeta-two, so the read-into-a-local workaround was removed.)
- **First-touch race.** The lazy init is not yet once-guarded against two fibers
  initializing the key simultaneously. Single-threaded use is unaffected; the
  guard is tracked in `plan/primavera-plan.md`.
- **Session scope** is not request scope. Sessions outlive a single request and
  are shared across a client's requests, so they need an owning, expiring,
  concurrency-safe store rather than a per-request binding — designed as the
  next increment (`plan/primavera-plan.md`), not shipped here.
