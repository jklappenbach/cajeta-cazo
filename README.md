# primavera

An opinionated enterprise framework for [Cajeta](https://github.com/jklappenbach/cajeta) —
roughly what Spring is to Java. primavera is built **on** the language's neutral
primitives (`@Inject`, `FiberLocal`, reflection); it ships **policy** —
component lifecycle, request/session scope, and (coming) a web request/session
model — so the core language stays small and other frameworks remain free to
innovate their own patterns.

> **Why a separate library?** Cajeta the *language* ships mechanism: dependency
> injection points (`@Inject`), ambient per-request state (`FiberLocal`),
> reflection. primavera ships an *opinion* about how to assemble those into
> enterprise services. If primavera's opinion isn't yours, you keep the primitives
> and build your own — nothing in the core forces primavera on you.

## Status — v0.1.0 (early)

| Capability | State |
|---|---|
| **Request scope** over `FiberLocal` (`RequestScope`, `ScopeMap`) | ✅ built, self-tested |
| Component model (`@Component` / `@Aspect` / lifecycle) — spec & docs | ✅ relocated here (`docs/AspectModel.md`); annotations are compiler-recognized today |
| Session scope (`SessionScope` + a session store) | ▢ designed, next increment |
| Web request/session model + pluggable executor | ▢ planned |
| primavera-side unit-test helpers (mock `@Request`/`@Session`, both executors) | ▢ planned (builds on [cajeta-unit](https://github.com/jklappenbach/cajeta-unit)) |

See [`plan/primavera-plan.md`](plan/primavera-plan.md) for the roadmap.

## Request scope

The load-bearing idea: request-scoped state is keyed on the **logical request**,
not on a thread or fiber. Because it rides a `FiberLocal` binding, one mechanism
serves both webserver concurrency models:

- **thread / fiber-per-request** — the request owns one fiber start to finish,
  so the scope *is* that fiber's binding. Cajeta's cheap fibers make this scale
  where OS-thread-per-request can't.
- **threadpool over completion ports** (epoll / io_uring / IOCP) — the request
  isn't pinned to a thread; its continuation hops pool threads as I/O completes.
  A thread-local bag would be a correctness bug here. The scope travels *with*
  the request — structured fan-out inherits the binding, an unstructured handoff
  carries it via a `FiberContext` snapshot — so it follows the request across
  thread hops and never leaks across requests.

```cajeta
import org.cajeta.primavera.context.RequestScope;

RequestScope.enter(() -> {
    // current() is live for the whole call tree — no parameter threading.
    if (RequestScope.has("com.app.Cart") == false) {
        RequestScope.store("com.app.Cart", #(heap Cart()));
    }
    Cart cart = (Cart) RequestScope.lookup("com.app.Cart");
    handle(request, cart);
});   // the bag — and every bean it owns — drops here
```

The bag (`ScopeMap`) **owns** its beans: when the request ends, the whole graph
of request-scoped instances drops with it.

## Build & test

Requires the Cajeta toolchain on `PATH` (or invoke the versioned binary
directly):

```
cajeta build    # compile to build/archive/org.cajeta.primavera-<version>.cja
cajeta test     # build + run the runtime self-tests (fails the build on any failure)
```

## Documentation

- [`docs/RequestScope.md`](docs/RequestScope.md) — request scope: model, ownership, concurrency.
- [`docs/AspectModel.md`](docs/AspectModel.md) — component model + AOP spec (relocated from the Cajeta stdlib docs; primavera is its home).
- [`plan/primavera-plan.md`](plan/primavera-plan.md) — roadmap.

## License

Apache-2.0.
