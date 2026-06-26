# Enterprise framework landscape — comparative analysis + primavera scope

> **Purpose.** Categorize the capability surface of Spring and its competitors,
> compare coverage, and decide — category by category — **what primavera should
> take on, defer, or cede.** This is the strategic scope map behind
> `plan/primavera-plan.md`.
>
> **Status:** living document, drafted 2026-06-25. Strategic taxonomy, not a
> version-pinned feature audit; framework specifics reflect a Jan-2026 view.

## How to read this — Cajeta's differentiators are the lens

Every "primavera stance" below is decided against four properties of Cajeta that
make it *unlike* the JVM/.NET incumbents. They are the reason primavera should not
just be "Spring, ported."

1. **AoT, compile-time, no reflection.** The DI/AOP substrate is resolved and
   codegenned at compile time (stdlib `cajeta.aot`). primavera should extend that
   model to *everything*: HTTP clients, config binding, serialization, validation,
   security policy, routing — all generated, none reflective. This is the
   Micronaut/Quarkus bet, and Cajeta is built for it natively.
2. **Borrow-checked, no GC.** Resource lifecycles (connections, pools, buffers) are
   explicit and deterministically dropped. This gives predictable latency (good for
   the ML-serving and embedded targets) but means every capability must define
   ownership, not assume a GC will clean up.
3. **Cheap fibers + `FiberLocal`.** Fiber-per-request scales; structured fan-out and
   `FiberLocal` already solve context propagation. Resilience, async, and
   cross-cutting context (tracing/security/tx) ride this instead of reactive-stream
   plumbing.
4. **One toolchain, three targets — embedded, enterprise, ML/DS.** Capabilities must
   be **modular and dead-code-eliminable**: link only what you use. primavera should
   be a set of opt-in capability modules over a lean core, not a monolith.

## Competitor roster

| Framework | Lineage / model | Why it matters here |
|---|---|---|
| **Spring Boot** | JVM, runtime reflection + proxies; the feature-completeness benchmark | The superset to measure coverage against |
| **Micronaut** | JVM, **compile-time** DI/AOP, GraalVM-native | Closest philosophical cousin; proves AoT enterprise DX |
| **Quarkus** | JVM, **build-time**, MicroProfile, K8s-native, dev services | The other AoT leader; best native + developer-experience story |
| **ASP.NET Core** | .NET, middleware pipeline, strong DI | The other mature, cross-platform enterprise stack; great resilience (Polly), gRPC, OpenTelemetry |
| **Go ecosystem** | stdlib `net/http` + à-la-carte libs (Gin/Echo, Wire, sqlc) | The "lean single binary, compose libraries" model — Cajeta's embedded/systems side |
| *also referenced* | Jakarta EE / MicroProfile (the standard), Helidon (Níma/virtual threads), NestJS (decorator DI, TS), Vert.x (reactive), Dropwizard (curated bundle) | Points of comparison in prose |

---

## Capability taxonomy + primavera stance

Coverage marks in the matrices: ● first-class · ◐ partial / via module · ○ library/DIY.

### A. Foundation

| Capability | Spring | Micronaut | Quarkus | ASP.NET | Go | primavera stance |
|---|:--:|:--:|:--:|:--:|:--:|---|
| DI / IoC container | ● | ● | ● | ● | ◐ | **core stdlib** (`@Component`/`@Inject`/`@Factory`) — done |
| AOP / interceptors | ● | ● | ● | ◐ | ○ | **core stdlib** (aspect weaving) — done |
| Externalized config | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** Type-safe, compile-bound `@Config`/`@Value`; layered sources (file/env/args); profiles. The missing primitive flagged earlier; serves enterprise *and* ML experiment-config. |
| Bean validation | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** Declarative constraints compiled to direct checks (no reflection); method-param + request-body validation. |
| Stereotypes (`@Service`/`@Repository`) | ● | ● | ● | ◐ | ○ | **TAKE — Tier 1.** Thin named roles over core `@Component`. |

### B. Web / API tier

> **The HTTP engine is the [`cajeta-http`](https://github.com/jklappenbach/cajeta-http)
> library, not primavera.** HTTP/WS/SSE live there (over the `cajeta.io.net`
> transport); primavera's job in this tier is *policy* — annotation-driven
> endpoints, routing-by-signature, and automatic serde-to-object — layered on
> cajeta-http's imperative `HttpServer`/`Router`/`WebSocket`.

| Capability | Spring | Micronaut | Quarkus | ASP.NET | Go | primavera stance |
|---|:--:|:--:|:--:|:--:|:--:|---|
| HTTP server + routing (`@RestServer`) | ● | ● | ● | ● | ● | **TAKE — Tier 1 (Phase 4), as policy over `cajeta-http`.** Annotation-routed endpoints + auto-serde over cajeta-http's server (which rides `cajeta.io.net`'s accept models); primavera does **not** ship the HTTP engine. The headline. |
| Content negotiation / serialization (JSON, proto) | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** Compile-time codecs (Cajeta already has `cajeta.codec`); no reflective (de)serialization. |
| Request validation / binding | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** Folds into §A validation. |
| OpenAPI / schema generation | ● | ● | ● | ● | ◐ | **TAKE — Tier 2.** Generate the spec at compile time from handler signatures (Micronaut model). |
| WebSocket / SSE | ● | ● | ● | ● | ◐ | **TAKE — Tier 2.** Streaming over fibers. |
| gRPC | ◐ | ● | ● | ● | ● | **TAKE — Tier 2.** Strong fit: codegen + fibers; matters for ML/service meshes. |
| GraphQL | ◐ | ◐ | ◐ | ◐ | ○ | **DEFER — Tier 3.** Niche; revisit on demand. |
| Server-side MVC / templating | ● | ◐ | ◐ | ● | ◐ | **CEDE.** API-first; SSR views are out of primavera's lane. |

### C. Outbound integration — adapters / clients for 3rd-party services

> The category the user called out explicitly. This is where AoT pays off most:
> a declarative client compiles to a direct call, no dynamic proxy.

| Capability | Spring | Micronaut | Quarkus | ASP.NET | Go | primavera stance |
|---|:--:|:--:|:--:|:--:|:--:|---|
| Declarative HTTP client (`@Client`/Feign/Retrofit) | ● | ● | ● | ◐ | ◐ | **TAKE — Tier 1.** Interface → generated typed client over cajeta-http's `HttpClient`; resilience (§E) wraps it. A flagship AoT differentiator. |
| Service discovery / client-side LB | ● | ● | ● | ◐ | ◐ | **DEFER — Tier 3.** Cloud-native; needs a registry. Provide the *seam*, not the impl. |
| Cloud-service SDK adapters (object store, queues…) | ◐ | ◐ | ● | ● | ◐ | **SEAM — Tier 2.** Define capability interfaces (e.g. `ObjectStore`, `Queue`); ship a couple, let others plug in. (Cajeta already has `cajeta-cloud-objectstore`, `cajeta-gossip`.) |
| Message-broker clients (Kafka/AMQP/JMS) | ● | ● | ● | ◐ | ◐ | **part of §D.** |
| Mail / SMS / push gateways | ● | ◐ | ◐ | ◐ | ○ | **CEDE to ecosystem libs.** Provide the client seam, not built-ins. |

### D. Integration / messaging / eventing

| Capability | Spring | Micronaut | Quarkus | ASP.NET | Go | primavera stance |
|---|:--:|:--:|:--:|:--:|:--:|---|
| Message brokers (Kafka, AMQP, JMS) | ● | ● | ● | ◐ | ◐ | **TAKE — Tier 2.** Producer/consumer over fibers + structured concurrency; one `@Listener` surface, pluggable broker. |
| Pub/sub + in-process events | ● | ● | ● | ● | ◐ | **TAKE — Tier 2.** Lightweight in-process event bus first. |
| Enterprise integration patterns (Camel/Spring Integration) | ● | ○ | ◐ | ○ | ○ | **CEDE.** Heavy DSL; a separate library if ever. |
| Streaming / CDC | ◐ | ◐ | ● | ◐ | ◐ | **DEFER — Tier 3.** |

### E. Resilience + concurrency (incl. retry semantics)

> The user's "retry semantics." Resilience4j / Polly / MicroProfile Fault Tolerance
> are the references. Cheap fibers make timeouts/bulkheads natural.

| Capability | Spring | Micronaut | Quarkus | ASP.NET | Go | primavera stance |
|---|:--:|:--:|:--:|:--:|:--:|---|
| Retry (backoff, jitter, predicates) | ◐ | ● | ● | ● | ◐ | **TAKE — Tier 1.** `@Retry` advice over the core aspect weaver; declarative policy. |
| Circuit breaker | ◐ | ● | ● | ● | ◐ | **TAKE — Tier 1.** Pairs with the declarative client (§C). |
| Bulkhead / rate limit / timeout | ◐ | ● | ● | ● | ◐ | **TAKE — Tier 2.** Timeouts/bulkheads are cheap with fibers. |
| Fallback | ◐ | ● | ● | ● | ◐ | **TAKE — Tier 1.** With retry/breaker. |
| Async / reactive model | ● | ● | ● | ● | ● | **TAKE — own model.** **Fibers + structured concurrency**, not reactive streams. The differentiator — no `Mono`/`Flux` ceremony. |
| Scheduling (`@Scheduled`/cron) | ● | ● | ● | ● | ◐ | **TAKE — Tier 2.** |
| Batch processing | ● | ◐ | ◐ | ◐ | ○ | **DEFER — Tier 3.** |

### F. Security

| Capability | Spring | Micronaut | Quarkus | ASP.NET | Go | primavera stance |
|---|:--:|:--:|:--:|:--:|:--:|---|
| AuthN: OAuth2 / OIDC / JWT | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** JWT/OIDC resource-server first (the modern default). |
| AuthN: mTLS / API keys / basic | ● | ● | ● | ● | ◐ | **TAKE — Tier 2.** |
| AuthN: SAML / LDAP | ● | ◐ | ◐ | ● | ○ | **DEFER — Tier 3.** Legacy enterprise; on demand. |
| AuthZ: RBAC + method security | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** `@Secured`/`@RolesAllowed` as aspect advice. |
| AuthZ: ABAC / policy engine | ◐ | ◐ | ◐ | ◐ | ○ | **DEFER — Tier 3.** |
| CORS / CSRF | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** Server-tier filters. |
| Secrets management | ◐ | ◐ | ● | ◐ | ◐ | **SEAM — Tier 2.** Interface over Vault/KMS/env; ship env + one provider. |
| Crypto / hashing utilities | ◐ | ◐ | ◐ | ● | ● | **stdlib, not primavera.** Belongs in core `cajeta.crypto`. |

### G. Data access + transactions

| Capability | Spring | Micronaut | Quarkus | ASP.NET | Go | primavera stance |
|---|:--:|:--:|:--:|:--:|:--:|---|
| Repository abstraction | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** Compile-time generated repositories (Micronaut Data model — no runtime proxies). `@Repository` stereotype already reserved. |
| ORM (full, Hibernate-style) | ● | ◐ | ● | ● | ○ | **CEDE the heavyweight ORM.** Favor explicit, lightweight SQL mapping (jOOQ/sqlc spirit) — fits no-GC + predictable cost. |
| Query / SQL builder | ◐ | ● | ◐ | ● | ◐ | **TAKE — Tier 2.** Type-checked query building. |
| Transactions (`@Transactional`) | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** Aspect-driven; `FiberLocal` carries the tx context — a natural fit. |
| Distributed tx / XA / sagas | ● | ◐ | ◐ | ◐ | ○ | **DEFER — Tier 3.** Sagas over messaging if/when. |
| Connection pooling | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** Owned, deterministically-dropped pool — borrow-checker fit. |
| Migrations (Flyway/Liquibase) | ● | ● | ● | ● | ◐ | **SEAM — Tier 2.** |
| NoSQL / cache stores (Redis, Mongo) | ● | ● | ● | ● | ◐ | **SEAM — Tier 2.** Client seams; ship Redis. |

### H. Observability + operations

| Capability | Spring | Micronaut | Quarkus | ASP.NET | Go | primavera stance |
|---|:--:|:--:|:--:|:--:|:--:|---|
| Metrics (Micrometer/Prometheus) | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** |
| Distributed tracing (OpenTelemetry) | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** Context propagation via `FiberLocal` — already solved at the substrate. A differentiator. |
| Structured logging | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** Over `cajeta-logging` (already a dependency intent). |
| Health checks / readiness | ● | ● | ● | ● | ◐ | **TAKE — Tier 1.** |
| Management/admin endpoints (Actuator) | ● | ● | ● | ● | ○ | **TAKE — Tier 2.** |

### I. Caching

| Capability | Spring | Micronaut | Quarkus | ASP.NET | Go | primavera stance |
|---|:--:|:--:|:--:|:--:|:--:|---|
| Cache abstraction (`@Cacheable`) | ● | ● | ● | ● | ◐ | **TAKE — Tier 2.** Aspect-driven; pluggable store. |
| Distributed cache (Redis/Hazelcast) | ● | ● | ● | ● | ◐ | **SEAM — Tier 2.** |

### J. Runtime / developer experience

| Capability | Spring | Micronaut | Quarkus | ASP.NET | Go | primavera stance |
|---|:--:|:--:|:--:|:--:|:--:|---|
| Native / AoT compilation | ◐ | ● | ● | ◐ | ● | **inherent.** Cajeta *is* AoT — primavera's structural advantage over Spring. |
| Live reload / dev mode | ● | ● | ● | ● | ◐ | **DEFER — Tier 3.** Tooling, not framework. |
| Dev services (auto Testcontainers) | ○ | ◐ | ● | ○ | ○ | **DEFER — Tier 3.** Nice-to-have; pairs with the test harness. |
| Serverless / FaaS | ● | ● | ● | ● | ◐ | **DEFER — Tier 3.** |

---

## primavera scope — consolidated tiers

**Tier 1 (the MVP enterprise stack — the credible-Spring-alternative bar):**
config + validation + stereotypes (§A); HTTP server + serialization + binding (§B);
declarative HTTP client (§C); retry / circuit-breaker / fallback (§E); fiber +
structured-concurrency async model (§E); JWT/OIDC + RBAC + CORS (§F); repositories +
`@Transactional` + connection pooling (§G); metrics + tracing + logging + health (§H).

**Tier 2 (rounding out):** OpenAPI, WebSocket/SSE, gRPC (§B); cloud/secret/store seams
(§C/§F/§G); messaging + in-process events (§D); bulkhead/ratelimit/timeout, scheduling
(§E); mTLS (§F); query builder, migrations, NoSQL seams (§G); admin endpoints (§H);
caching (§I).

**Tier 3 (on demand):** GraphQL, batch, EIP/Camel, streaming/CDC, SAML/LDAP, ABAC,
distributed tx/sagas, service discovery, dev services, serverless.

**The through-line:** for each Tier-1/2 item, the deliverable is *compile-time
generated, fiber-native, ownership-explicit, and dead-code-eliminable*. That is the
single sentence that makes primavera not-just-Spring.

## Deliberately ceded (not primavera's lane)

- **Heavyweight ORM** (Hibernate-style runtime mapping) — favor explicit lightweight SQL.
- **Server-side MVC / templating** — API-first.
- **Enterprise integration DSLs** (Camel/Spring Integration).
- **Crypto primitives** — belong in core stdlib (`cajeta.crypto`), not the framework.
- **Concrete cloud/mail/SMS SDKs** — primavera ships the *seam*; ecosystem libs ship
  the adapters (matches Cajeta's `cajeta-*` multi-repo model).

## Open decisions

1. **Capability-module granularity.** One `primavera-web`, `primavera-data`,
   `primavera-security`… set of repos (à la Cajeta's split) vs one library with
   DCE-gated features? Leaning multi-module to honor the embedded/DCE target.
2. **Async surface.** Commit fully to fibers + structured concurrency and offer *no*
   reactive-streams compatibility, or provide an adapter? Leaning fibers-only.
3. **Data tier depth.** How far past "generated repositories + SQL mapping" to go
   before it becomes an ORM we said we'd cede.
4. **Seam vs battery.** For each "SEAM" item, which one default adapter ships in-tree
   so the capability is usable out of the box (e.g. Redis for cache, env for secrets).
5. **Config/value injection spec** is the first Tier-1 gap with no doc yet — likely the
   next spec after `@RestServer`.
