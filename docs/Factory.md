# Factory providers (`@Factory`)

How a type that isn't a directly-instantiable `@Component` enters the DI graph:
third-party/unowned types, instances that need **assisted (non-injected)
constructor arguments**, and instances that need **initialization beyond the
constructor**. One `@Factory` class holds one method per provided type.

> **`@Factory` is part of the CORE DI substrate — its normative spec is the
> cajeta stdlib `docs/specification/lang/AspectModel.md` (§ `@Factory`), not
> primavera.** primavera does not own `@Factory` any more than it owns
> `@Component`. This document is kept as the **design rationale** — why a factory
> at all, the four soundness rules, and the `@Bean`/cast rejections — that fed the
> stdlib spec. Status: design recorded 2026-06-25; implementation pending in the
> compiler.

## Why a factory at all

`@Component` + `@Inject` (`docs/AspectModel.md`) covers the common case: a class you
own, fully resolvable from the graph, constructed canonically by the compiler. It
**cannot** express three things, and a method body enclosing construction provides
all three at once:

1. **Third-party / unowned types** — you can't put `@Component` on a class you can't
   edit; a factory method's body calls its constructor for you.
2. **Assisted arguments** — constructor parameters supplied by the caller at use
   time (a tenant id, a hyperparameter), not resolved from the graph.
3. **Initialization beyond the constructor** — two-phase init, builder chains, an
   `.open()` call — work a constructor can't do.

```cajeta
@Factory class ConnectionFactory {
    @Inject Pool pool;                              // factory's own collaborator (field)
    @Singleton Connection make(@Inject Clock clk, String tenantId) {
        Connection c = heap Connection(pool, clk, tenantId);  // injected + assisted
        c.open();                                   // init the ctor can't express
        return c;                                   // fresh, owned
    }
}
```

## Coexistence with `@Inject` constructors

> **DECISION (recommended; flip-point noted).** `@Factory` **coexists** with
> `@Inject` construction rather than replacing it. `@Inject` constructor/field
> injection stays the terse default for fully graph-resolvable, owned classes; a
> `@Factory` method is required **only** when you need one of the three things above.
> The alternative — mandating a factory method for *every* constructed component
> (Guice-pure) — was rejected as boilerplate for the trivial case. If uniform
> explicitness is later judged more valuable than terseness, this is the one rule to
> revisit.

A given type is provided by **either** a `@Component` constructor **or** a `@Factory`
method, never both — both providing the same type is an ambiguity compile error, on
the same rules as two `@Component`s of one interface (`docs/AspectModel.md` §"Qualifying
an injection"). `name = "..."` qualifiers and the *unqualified-is-default* rule carry
over unchanged.

## The four rules

### R1 — resolution keys on the method *signature*, never the body

The DI graph is built and checked (missing / cyclic / ambiguous) from signatures
alone: **return type = the provided type; `@Inject`-marked params = the dependency
edges.** The body is opaque to resolution — it matters only to codegen. This is what
keeps the compile-time-graph guarantee intact even though the body is arbitrary code.

### R2 — fresh-owned return; the framework owns caching

A `@Factory` method returns a freshly-`heap`-allocated **owned** value (`#T`). It is an
ordinary owned-return method, so the borrow checker's existing "returned value is
owned, not a borrow" discipline applies with no special case.

> **The body must not cache or manage its own lifetime.** Returning the same instance
> twice aliases an owned value and breaks single-owner. Scope is declared on the
> method (R4); the *generated* accessor memoizes a singleton and owns its drop. The
> body always returns fresh.

### R3 — injected vs assisted parameters

This is the core distinction. In a factory method, **`@Inject`-marked params are
graph-resolved; unmarked params are assisted** (supplied by the caller). The presence
of an assisted param changes how the method is consumed:

| Method shape | How consumers use it |
|---|---|
| **all params `@Inject`** (or none) | a *provider* — sites just `@Inject Connection`; the generated `get_/make_Connection()` calls the factory method, threading the injected args |
| **≥1 assisted param** | an *assisted factory* — sites `@Inject ConnectionFactory` and call `make(tenantId)` explicitly, passing the assisted args; the compiler threads any `@Inject`-marked params at that call site |

Guidance: a collaborator the factory *always* needs (`Pool` above) is cleaner as an
`@Inject` **field** of the factory class than as an `@Inject` method param — fields are
resolved once at factory construction; `@Inject` method params are threaded per call.
Reserve `@Inject` method params for per-call injected deps.

### R4 — scope is declared on the method

`@Singleton` / `@Transient` / etc. annotate the factory method, fixing the lifetime of
what it provides in one legible place (a deliberate contrast with `@Inject(allocate=…)`
declaring scope at each *site* — see `docs/AspectModel.md`). The generated accessor
implements the scope: memoize for singleton, fresh per call for transient. (Trade-off:
this gives up "same type, different lifetime per consumer"; for the enterprise target,
one-declared-scope legibility is the better default.)

## Codegen sketch

```cajeta
// @Factory class is itself a singleton component (it can @Inject collaborators).
static ConnectionFactory __factory_ConnectionFactory = /* get_…() lazy singleton */;

// Provider method (all-injected) — product is auto-resolvable:
static Connection get_Connection() {                 // @Singleton provider
    if (__singleton_Connection == null)
        __singleton_Connection =
            __factory_ConnectionFactory.make(get_Clock());   // injected args threaded
    return __singleton_Connection;
}

// Assisted factory — consumer injects the factory, calls it:
//   @Inject ConnectionFactory cf;  ...  Connection c = cf.make(tenantId);
// compiler threads the @Inject param, leaves the assisted one to the caller:
//   c = __factory_ConnectionFactory.make(get_Clock(), tenantId);
```

No reflection, direct calls — identical discipline to `get_X()`/`make_X()` for plain
components.

## Third-party types — the gap this closes

A type you can't annotate enters the graph through a one-line factory body:

```cajeta
@Factory class Persistence {
    @Singleton Database db(@Inject Config cfg) {
        return heap PostgresClient(cfg.url);   // PostgresClient is vendored, un-annotatable
    }
}
// consumers just @Inject Database — they never see the factory.
```

The factory body is the canonical construction site the compiler couldn't otherwise
reach; everything downstream (`@Inject Database`) is unchanged.

## Testing

Factory providers participate in the override model (`docs/Testing.md`) by type: a test
`@Factory` (or `@TestComponent`) providing the same type replaces the production
provider at test compile time. Assisted factories are injected like any component, so a
test can supply a fake factory directly (Layer 2) or override by type (Layer 1).

## Rejected

- **Spring-style `@Bean` methods** (producer methods scattered on `@Configuration`
  classes, free-floating). Folded into the single `@Factory`-class-with-methods shape:
  one home per provided type, no second free-form configuration dialect.
- **Construction via overloaded cast** (`MyObj m = (MyObj) factory;`). A cast means
  "reinterpret an existing value," not "allocate"; it has nowhere to carry assisted
  arguments, and it hides the binding and scope behind punctuation. A named method call
  carries args and names the provider. Conversion operators are for value conversion,
  not construction.
- **Mandatory factories for all construction** (Guice-pure). Boilerplate for the
  fully-injectable case; see the coexistence decision above.

## Open decisions

- **Call-site threading of `@Inject` method params on assisted factories** (R3) is mild
  compiler magic — the consumer writes `make(tenantId)` and the compiler fills the
  `@Inject` slots. Acceptable, but the alternative (assisted methods take *only*
  assisted params; all injected deps are factory fields) is simpler to reason about.
- **Whether `@Factory` methods may be `static`** (no factory-instance state) as a
  lighter form when the factory needs no injected collaborators.
- **Multibinding** (`@Inject List<A>` collecting every provider of `A`) is still
  unaddressed here — tracked with the Q2 interface-ambiguity discussion, not this spec.
