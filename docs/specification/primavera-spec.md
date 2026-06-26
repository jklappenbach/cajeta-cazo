# primavera — specification

The canonical specification for primavera, Cajeta's opinionated enterprise
framework. This document defines the framework's surface; per-topic deep dives
live in the sibling docs it links. For *what* primavera takes on vs cedes and
*why* (versus Spring/Micronaut/Quarkus/.NET/Go), see
[`EnterpriseFrameworkLandscape.md`](EnterpriseFrameworkLandscape.md).

> **Status (2026-06).** Request scope is shipped; the rest is specified here and
> tracked to completion by the cajeta project's `primavera-completion` plan.
> Sections are marked **✅ shipped**, **◆ per spec** (specified, not built), or
> **▢ planned** (outlined).

## 1. What primavera is, and where it sits

primavera is **policy** assembled over three lower layers it does not own:

```
primavera (this spec)   — enterprise POLICY: scopes, web endpoints, config,
                          resilience, security, data, observability
   ├─ cajeta-http        — HTTP/1.1·2·3, WebSocket, SSE (a library)
   ├─ cajeta.io.net      — transport: sockets, reactor, TLS (stdlib)
   └─ cajeta.aot         — DI substrate: @Component/@Inject/@Factory + aspects (stdlib)
      + cajeta.concurrent.FiberLocal — ambient per-request context (stdlib)
```

If primavera's opinion isn't yours, you keep all three lower layers and build your
own framework on the same substrate — nothing forces primavera.

### Design principles

1. **Compile-time, no reflection.** Every primavera mechanism — config binding,
   validation, routing, security policy, declarative clients, repositories — is
   resolved and code-generated at compile time over the AoT DI substrate. No
   runtime proxies, no reflective (de)serialization.
2. **Fiber-native, no function coloring.** Concurrency is cheap fibers +
   structured concurrency + `cajeta.io`'s `Stream<T>`; there is no reactive-streams
   ceremony and no sync/async split. A handler is ordinary blocking-looking code.
3. **Ownership-explicit.** Resources (pools, connections, scope bags) are owned and
   deterministically dropped — predictable latency, no GC, fits the ML-serving and
   embedded targets.
4. **Modular / dead-code-eliminable.** Capabilities are opt-in; a service links
   only what it uses. See §2.
5. **Policy over substrate.** primavera adds *opinion*; the *mechanism* stays in
   the stdlib/cajeta-http layers, reusable by anyone.

## 2. Module structure

primavera is a set of capability modules over a lean core, so a service pulls only
what it needs (the embedded/DCE target):

| Module | Provides |
|---|---|
| `primavera-core` | scopes, stereotypes, profiles, config, validation, lifecycle glue |
| `primavera-web` | `@Rest`/`@RestServer` endpoints + auto-serde (over cajeta-http) |
| `primavera-resilience` | `@Retry`/`@CircuitBreaker`/`@Fallback`/… |
| `primavera-security` | authn/authz, CORS, secrets seam |
| `primavera-data` | `@Repository` generation, `@Transactional`, pooling |
| `primavera-observability` | metrics, tracing, health, management endpoints |
| `primavera-client` | `@Client` declarative outbound clients |
| `primavera-test` | test harness (on cajeta-unit) |

> **Open decision.** Multi-repo modules (à la Cajeta's `cajeta-*` split) vs one
> library with DCE-gated features. Leaning multi-module to honor the DCE target;
> §16.

## 3. Component model, stereotypes & scopes

### 3.1 Substrate (stdlib, recap)

`@Component` / `@Inject` / `@Factory`, the compile-time graph, identity scopes
(`SINGLETON` / `OWNER_SCOPE` / `CALL_SCOPE` / `TRANSIENT`), and `@PostConstruct` /
`@PreDestroy` are **core language** (`cajeta.aot`). primavera consumes them; see the
stdlib `AspectModel.md` and primavera's [`AspectModel.md`](../AspectModel.md) pointer
and [`Factory.md`](../Factory.md).

### 3.2 Stereotypes — **per spec**

Named roles over `@Component`, identical DI behavior, carrying documentary intent
and a hook for role-specific defaults:

- `@Service` — business-logic component.
- `@Repository` — data-access component (see §11; gets generated query methods).
- `@Controller` is **not** a stereotype — web entry points are `@Rest` endpoints (§6).

### 3.3 Request scope — **✅ shipped**

The per-request bag over `FiberLocal`, correct under both fiber-per-connection and
event-driven executors. Full spec: [`RequestScope.md`](../RequestScope.md). A
request-scoped component resolves through `RequestScope` get-or-create rather than
the singleton path (wired at the web boundary, §6).

### 3.4 Session scope — **per spec**

A session outlives a single request and is shared across a client's requests, so it
is **not** a `FiberLocal` binding — it is an owning, id-keyed, expiring,
concurrency-safe store:

```cajeta
@Component class SessionStore {                 // singleton
    Session getOrCreate(String id);             // by session id
    Optional<Session> get(String id);
    void invalidate(String id);                 // drops the session + its bean graph
    // expiry: idle TTL + absolute TTL; concurrency-safe (Lock/LockGuard);
    // timestamps from Clock.millisTime().
}
@Component class SessionScope {                  // ambient access to the current session
    static boolean isActive();
    static ScopeMap current();                   // resolves the bound id → store
    // binds the session ID (a cheap owned String) via FiberLocal for the request
    // extent; the session map's lifetime is the store's, never the request binding's.
}
```

> **BLOCKER.** A correct *evicting* store must **own** the session bags (so an
> invalidated session's beans free) and remove them on expiry. Cajeta collections
> are non-owning today; this needs the compiler-level owning-collection work
> (stdlib `owning-collections` plan). Session scope is blocked on it — the API is
> specified, the implementation waits.

### 3.5 Deployment profiles — **per spec**

`@Profile("prod"|"test"|"ci"|…)` on a component includes it only when the active
`--profile` matches; profile-neutral components always apply. One active profile per
compilation; default `prod`, test driver defaults `test`. Resolution and ambiguity
rules carry over from the substrate.

## 4. Configuration & value injection — **per spec** (the Tier-1 gap)

The missing Tier-1 primitive, serving enterprise externalized config *and* ML/DS
experiment config. Compile-time-bound where possible, runtime sources behind it.
**Full spec: [`Configuration.md`](../Configuration.md)** (sources/precedence,
the compile-time-vs-runtime contract, frozen-mode for embedded, interpolation +
secrets, conditional wiring, ML experiment composition, the effective-config dump).

```cajeta
@Component class Server {
    @Value("server.port") int32 port;                 // single value
    @Value("server.host", default="0.0.0.0") String host;
}

@Config(prefix="db") class DbConfig {                  // typed, validated group
    String url;
    int32 poolSize = 8;                                // defaults in the type
    Duration idleTimeout = Duration.ofSeconds(90);
}
@Component class Repo { @Inject DbConfig cfg; }
```

- **Layered sources**, highest-precedence first: CLI args → environment →
  profile files (`application-<profile>.toml`) → `application.toml` → type defaults.
- **Type-safe binding** — values bind to typed fields; coercion failures and
  missing-required-without-default are **compile-time** errors where the key set is
  statically known, runtime-validated otherwise.
- **Conditional wiring** — `@Requires(property="feature.x", value="on")` includes a
  component only when config matches: build-time conditional wiring (the Micronaut
  `@Requires` analog) that closes most of the "flexibility" gap without a runtime
  container.
- **Secrets** are a **seam** — `@Value` can resolve from a `SecretSource`
  (env/Vault/KMS); ship env + one provider (§16).

## 5. Validation — **per spec**

Declarative constraints compiled to **direct checks** (no reflection), applied to
request bodies (§6) and method parameters:

```cajeta
@Config(prefix="db") class DbConfig {
    @NotBlank String url;
    @Min(1) @Max(256) int32 poolSize = 8;
}
@Rest @Post("/orders")
Order create(@Valid @Body Order o) { ... }            // 400 with field errors on violation
```

Constraints: `@NotNull`/`@NotBlank`/`@NotEmpty`, `@Min`/`@Max`/`@Size`,
`@Pattern`, `@Email`, `@Past`/`@Future`, plus user-defined constraints. Method-level
validation runs as aspect advice; web-layer validation maps violations to a
structured 400.

## 6. Web / endpoint model — **per spec** (over cajeta-http)

primavera's headline policy: annotation-driven, signature-typed endpoints with
automatic serde, over cajeta-http's imperative `HttpServer`/`Router`. **primavera
does not ship the HTTP engine.** (Protocols, framing, executors: cajeta-http +
cajeta.io.net.)

```cajeta
@RestServer(port=8443, tls="server")               // host: assembles declared endpoints
class ApiServer {}

@Rest("/orders")                                    // an endpoint component (request-scoped beans resolve here)
class OrderEndpoints {
    @Inject OrderService svc;

    @Get("/{id}")  Order get(@Path int64 id) { ... }              // req/resp, materialized
    @Post()        void create(@Valid @Body View<Order> o) { ... } // zero-copy view body
    @Get()         Stream<Order> list(@Query("since") int64 t) { ... } // server-push stream (SSE/chunked)
}

@WebSocket("/feed")
Stream<Event> feed(in: Stream<Command>) { ... }     // duplex; interaction shape from signature
```

- **Annotation = protocol + route; signature = interaction shape + serde mode.** A
  `Stream<T>` return is server-push; `(Stream<In>) -> Stream<Out>` is duplex; `(Req)
  -> Resp` is request/response. `@Body View<T>` is zero-copy (borrowed,
  request-lifetime); `@Body T` materializes an owned value (§ data plane in
  `cajeta.io`'s `Stream<T>`/views).
- **Auto-serde** — bodies and return values (de)serialize via compile-time codecs
  (`cajeta.codec`); content negotiation by `Accept`/`Content-Type`.
- **Binding** — `@Path`, `@Query`, `@Header`, `@Body`, with `@Valid` (§5).
- **Request-scope binding** — each endpoint invocation runs under a fresh
  `RequestScope` (§3.3); request-scoped components resolve per request.
- **`@RestServer`** is the *host* stereotype: it assembles declared endpoints, picks
  the cajeta.io.net accept model, and stands up the cajeta-http server. There is no
  single "server class" you write imperatively — endpoints are the unit.
- **Filters** (CORS/CSRF/auth) are cajeta-http middleware configured by primavera
  security (§8).

## 7. Resilience — **per spec**

Aspect-driven declarative policies (Resilience4j/Polly analog), fiber-native
(timeouts/bulkheads are cheap with fibers):

```cajeta
@Retry(max=3, backoff="100ms", multiplier=2.0, jitter=true, on=[IoException.class])
@CircuitBreaker(failureRate=0.5, window=20, openFor="30s")
@Fallback("cachedQuote")
Quote fetchQuote(String symbol) { ... }
Quote cachedQuote(String symbol) { ... }            // fallback signature matches
```

Surface: `@Retry`, `@CircuitBreaker`, `@Fallback`, `@Bulkhead(max=N)`,
`@RateLimit(perSecond=N)`, `@Timeout("2s")`. Each is `@Around` advice over the core
aspect weaver; composes with the declarative client (§9). `@Order` controls
stacking.

## 8. Security — **per spec**

Compile-time-resolved authn/authz; no reflective security.

- **AuthN** — JWT/OIDC resource-server first (validate bearer tokens, extract
  claims); mTLS and API-key/basic as Tier-2; SAML/LDAP deferred. A `SecurityContext`
  rides `FiberLocal` (request principal, roles).
- **AuthZ** — method security as aspect advice: `@Secured`, `@RolesAllowed("admin")`,
  `@PreAuthorize(expr)`; ABAC/policy-engine deferred.
- **Web filters** — CORS, CSRF, security headers as cajeta-http middleware (§6).
- **Secrets** — the `SecretSource` seam (§4).
- **Crypto primitives** are **not** primavera's — they belong in stdlib
  (`cajeta.hash` / a future `cajeta.crypto`); primavera consumes them.

## 9. Outbound clients — **per spec** (over cajeta-http)

Declarative typed HTTP clients — an interface compiles to a direct call over
cajeta-http's `HttpClient`, no dynamic proxy (the AoT flagship):

```cajeta
@Client("https://payments.internal")
interface PaymentClient {
    @Post("/charge") @Retry(max=2) @CircuitBreaker(...)
    ChargeResult charge(@Body ChargeRequest req);
}
@Component class Checkout { @Inject PaymentClient payments; }
```

Resilience (§7) annotations wrap client methods. **Cloud/store/broker adapters are
seams** — primavera defines capability interfaces (`ObjectStore`, `Queue`,
`SecretSource`); concrete SDKs ship as ecosystem libraries, not built-ins (§16).

## 10. Async & streaming model — **per spec**

No reactive streams. primavera uses the stdlib **`Stream<T>`** (fiber-backed,
Flow-like, no coloring; backpressure via bounded channels) and structured
concurrency. Endpoint streaming (§6), messaging (Tier-2), and SSE are all the same
`Stream<T>`. See `cajeta.io`'s `Io.md` for the substrate.

## 11. Data & transactions — **per spec**

- **`@Repository`** — compile-time **generated** repositories (Micronaut Data
  model; no runtime proxies). Derived query methods (`findById`, `findByEmail`),
  explicit `@Query` for the rest.
- **`@Transactional`** — aspect-driven boundary; the transaction context rides
  `FiberLocal` (a natural fit — no thread-local hazard under the event-driven
  executor). Propagation (`REQUIRED`/`REQUIRES_NEW`); rollback rules.
- **Connection pooling** — an owned, deterministically-dropped pool (borrow-checker
  fit).
- **Migrations / NoSQL / cache stores** — seams; ship one default each (§16).
- **Heavyweight ORM is ceded** (§15) — favor explicit, lightweight SQL mapping.

## 12. Observability — **per spec**

- **Metrics** — `@Timed`/`@Counted` advice + a Micrometer-style registry
  (Prometheus export).
- **Tracing** — OpenTelemetry; **context propagation is already solved** by
  `FiberLocal` (the differentiator — spans follow the request across executor hops).
- **Logging** — structured, over `cajeta-logging`.
- **Health/readiness** + **management endpoints** (Actuator analog) as `@Rest`
  endpoints under a management prefix.

## 13. Caching — **per spec**

`@Cacheable`/`@CacheEvict`/`@CachePut` aspect advice over a pluggable `CacheStore`
(ship an in-memory/Caffeine-style default; Redis/distributed as a seam).

## 14. Testing — **✅ partly / per spec**

Full model: [`Testing.md`](../Testing.md). Four override layers (`@TestComponent`
compile-time swap ✅; direct construction ✅; `@Mock` codegen planned; the core
`@Inject` override seam shipped in stdlib), plus `withRequestScope` seeding and
running a handler under either executor. Built on
[cajeta-unit](https://github.com/jklappenbach/cajeta-unit).

## 15. Deliberately ceded (not primavera's lane)

Heavyweight ORM (Hibernate-style); server-side MVC/templating (API-first);
enterprise-integration DSLs (Camel/Spring Integration); crypto primitives (→
stdlib); concrete cloud/mail/SMS SDKs (primavera ships the *seam*, ecosystem ships
adapters). Rationale: [`EnterpriseFrameworkLandscape.md`](EnterpriseFrameworkLandscape.md) § ceded.

## 16. Open decisions

1. **Module granularity** — multi-repo `primavera-*` vs one DCE-gated library (§2).
2. **Async surface** — fibers-only, no reactive-streams adapter? Leaning yes.
3. **Data depth** — how far past "generated repositories + SQL mapping" before it
   becomes the ORM we ceded.
4. **Seam defaults** — which one adapter ships in-tree per seam (Redis for cache,
   env for secrets, …) so each capability is usable out of the box.
5. **`@Inject(allocate=…)` → `@Inject(scope=…)`** naming, and whether to allow a
   component-level **default scope** that sites may override (legibility vs
   site-flexibility).
6. **`@Primary`** — keep the implicit "unqualified component is the default", or add
   an explicit `@Primary`? And **multibinding** (`@Inject List<A>`) for plugin sets.
