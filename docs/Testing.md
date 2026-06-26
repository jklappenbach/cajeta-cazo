# Testing primavera components

How you supply a mock, fake, or stub in place of a DI-resolved component, bean, or
scoped instance — under unit and integration tests alike.

> **Status.** Layer 1 (`@TestComponent` / `@Profile`) is **shipped**. Layer 2
> (direct construction) works **today** for any constructor-injected component.
> Layer 4 (the per-test `@Inject` override seam) is a **shipped core stdlib**
> mechanism (singleton/class-typed; `docs/DI-override-hook.md`), driven via
> cajeta-unit's `TestContext`. Layer 3 (`@Mock` codegen) and the `withRequestScope`
> seeding helper are **planned** (`plan/DI-request-scope-and-test-enablers.md`).

## The one idea: overrides are compile-time graph substitutions

primavera's DI is AoT (`docs/AspectModel.md`): `@Inject Database db` compiles to a
direct call — `u.db = get_Database()` — with **no runtime container**. There is no
live registry to re-bind, so a test override cannot be "mutate the container before
the test." Instead an override is **a different generated graph, selected when you
compile through the test driver** (`--profile=test`), plus — for per-test behaviour
— runtime configuration of a compile-time-fixed double.

This yields four layers, at four granularities. Reach for the **coarsest one that
fits**; they compose.

| Layer | Granularity | Mechanism | Status |
|---|---|---|---|
| 1 | whole test build | `@TestComponent` / `@Profile("test")` | shipped |
| 2 | one SUT instance | direct constructor call (bypass DI) | works today |
| 3 | per-test behaviour | `@Mock` codegen (configurable double) | planned |
| 4 | per-test, DI-resolved dep | core `@Inject` override seam (`TestContext.bind`) | shipped (stdlib; singleton/class-typed) |

Cross-cutting both is **scope seeding** (`withRequestScope`, §6) for request/session
state rather than components.

## Layer 1 — whole-build type swap (`@TestComponent`)

The primary, shipped mechanism (`docs/AspectModel.md` §"Test-target overrides"):

```cajeta
@TestComponent public class FakeDatabase implements Database { ... }
```

At test compile time the graph resolver substitutes `FakeDatabase` for the real
`@Component … implements Database`. The generated `get_Database()` / `make_Database()`
now construct the fake, so **every injection site that asked for `Database`
transparently gets it — nothing changes at any call site.** Non-test builds ignore
`@TestComponent` entirely (no compiled code, no graph entry).

Rules that carry over from `@Component`:

- **Override is by declared type.** A `@TestComponent` replaces any profile-neutral
  or profile-matched `@Component` of the same type.
- **`name = "..."` still disambiguates** when several doubles share a type.
- **Does not combine with `@Profile`** (compile error — use one or the other). For a
  whole alternate graph, use `@Component @Profile("test")` declarations instead.

Use Layer 1 for *"the entire test binary uses the in-memory implementation."* Its
limit is granularity: it is **one double type for the whole compilation**, so on its
own it cannot give test A a DB-returns-empty and test B a DB-throws (see Layers 3–4).

## Layer 2 — direct construction (no framework)

The most robust per-test override needs no DI machinery at all. A constructor-injected
component (`docs/AspectModel.md` §"Component registration") can be built by hand, with
fakes passed positionally:

```cajeta
OrderService svc = heap OrderService(fakeDb, fakeMetrics);   // no get_X, no graph
```

The `@Inject` annotations are inert when you call the constructor yourself: you are the
caller that supplies the arguments. This is the cleanest, most isolated unit-test
override that exists, and it is available **today**.

> **Guidance.** Prefer Layer 2 for unit tests; reserve Layer 1 for integration-style
> tests that want the real graph with one seam replaced. This is the concrete reason
> to favour **constructor injection over field injection** in testable code: a
> field-injected dependency has no such hatch — only DI can populate it.

## Layer 3 — per-test behaviour (`@Mock`, planned)

Mockito-style stubbing that differs per test method (`when().thenReturn()` here,
`.thenThrow()` there) cannot use runtime proxies in an AoT language. The planned
`@Mock` codegen (`plan/DI-request-scope-and-test-enablers.md` §2.4) splits the problem:

- **compile time** — the compiler synthesizes a `Mock<T>` class implementing `T`'s
  interface (records calls, matches `any()`/`eq()`/`argThat()`, returns stubs). It is
  registered as a `@TestComponent`, so it drops into the graph where the prod
  component was (the gomock / mockall model — generated code, not a runtime proxy).
- **runtime, per test** — you configure *behaviour* on the instance:
  `when(m.find(id)).thenReturn(user)`, `verify(m, times(2))`.

So per-test variation comes from **runtime-configuring a compile-time-fixed mock
type**, never from re-binding the graph.

**Until `@Mock` lands,** the bridge is a hand-written **configurable fake** registered
as a `@TestComponent` — a fake exposing knobs each test sets in `@BeforeEach`:

```cajeta
@TestComponent public class FakeClock implements Clock {
    private int64 now = 0;
    public void setNow(int64 t) { this.now = t; }   // per-test knob
    public int64 millisTime() { return this.now; }
}
```

Because a `@TestComponent` singleton is the same instance every site sees, a test can
resolve it (or inject it) and configure it before exercising the SUT.

## Layer 4 — per-test override of a DI-resolved dependency (core seam, shipped)

Overriding a dependency resolved *through DI* (a deeply nested, field-injected
component) with a different double per test — without a global `@TestComponent` — is
handled by a **core stdlib seam**, not a primavera invention. It is the `@Inject`
runtime override hook (stdlib `docs/DI-override-hook.md`):

- **Test-build-only.** Gated on `--profile=test`; emitted in `ComponentInjectMethod`.
  Production builds emit the unchanged static path — zero cost, registry not even
  linked. (Efficiency preserved.)
- **Keyed on the type's `reflect.Class` pointer identity** (the same global `T.class`
  lowers to) — no string compare. The compiler emits a `select` at each overridable
  `@Inject` site: `ovr = __cajeta_inject_override_get(&T#ClassObject); dep = ovr ?? dep`.
- **Driven from cajeta-unit** via `org.cajeta.unit.TestContext`:

```cajeta
TestContext.bind(Database.class, fakeDb);    // any @Inject Database now resolves to fakeDb
try { handler.handle(req); assert(...); }
finally { TestContext.clear(); }             // next test unaffected
```

> **v1 limits (stdlib).** Singleton-mode, **class-typed** `@Inject` fields only (mock by
> subclassing and overriding virtuals; the substituted pointer dispatches through the
> subclass vtable). Interface-typed fields (24-byte fat pointers) and OwnerScope /
> Transient sites are **not yet overridable** — deferred in the stdlib hook. The
> registry borrows; the test keeps its mock alive for the graph's use.

primavera's contribution at this layer is policy, not mechanism: `@TestComponent`
(compile-time, Layer 1) and request-scope seeding sit *above* this core seam; the seam
itself is the language's, so any framework or harness can use it.

## Scope seeding (`withRequestScope`, planned)

Orthogonal to component overrides: a request-handler test must also enter a request
(or session) scope and seed it (`plan/DI-request-scope-and-test-enablers.md` §2.5),
built over the shipped `RequestScope` (`docs/RequestScope.md`):

```cajeta
withRequestScope(seed -> seed.store("com.app.Session", fakeSession), () -> {
    Response r = handler.handle(req);   // runs under a real request scope
    assert(r.status == 200);
});
```

Components are swapped by Layers 1/3/4; *scoped state* is seeded here.

## Putting it together — a request-handler test

```cajeta
// Layer 1: @TestComponent FakeRepository swapped in at compile time.
// Layer 4: per-test gateway double via the core override seam.
// Seeding: a faked session in request scope.
TestContext.bind(PaymentGateway.class, declinedGateway);
try {
    withRequestScope(seed -> seed.store("com.app.Session", fakeSession), () -> {
        Response r = checkoutHandler.handle(req);
        assert(r.status == 402);
    });
} finally { TestContext.clear(); }
```

## Relationship to the test framework

These layers are primavera-side capabilities consumed by [cajeta-unit](https://github.com/jklappenbach/cajeta-unit):
`@Test` discovery, `@BeforeEach`/`@AfterEach` (where per-test doubles are configured),
and `@Profile`/`@TestComponent` contexts. The dependency-ordered build-out lives in
`plan/DI-request-scope-and-test-enablers.md`; this document specifies the override
*model* those phases implement.
