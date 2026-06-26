# Component model + AOP — moved to the cajeta stdlib

> **The DI substrate and aspect weaving are CORE language now, not primavera.**
> `@Component`, `@Inject`, `@Factory`, the compile-time DI graph, the
> identity-based scopes (`SINGLETON` / `OWNER_SCOPE` / `CALL_SCOPE` /
> `TRANSIENT`), lifecycle hooks (`@PostConstruct` / `@PreDestroy`), and aspect
> weaving (`@Aspect` and the advice family) are specified in the **cajeta
> stdlib**:
>
> **`docs/specification/lang/AspectModel.md`** in the cajeta language repo
> (`package cajeta.aot`).

## Why it lives in the stdlib, not here

`@Inject` is the consumption end of a DI graph edge; `@Component` / `@Factory` are
the production end. A core `@Inject` resolving against library-defined producers
it cannot see or check is incoherent, and the AoT guarantees (compile-time graph,
ownership soundness, dead-code reachability) are all compiler-resident. The
substrate is **indivisible** — so it is core, and every library speaks one DI
vocabulary with no framework dependency (no Java-style
`@Inject`/`@Autowired`/`@Resource` fragmentation). primavera does **not** own
`@Component`; it is the *policy layer* built on the stdlib substrate.

## What primavera adds on top (policy)

primavera is the opinion assembled over the core substrate:

- **Request / session scope** — non-identity scopes keyed on the logical request,
  over `cajeta.concurrent.FiberLocal`. See [`RequestScope.md`](RequestScope.md).
- **Web model** — `@RestServer` / `@Rest` annotation-driven endpoints + auto-serde,
  layered on the [`cajeta-http`](https://github.com/jklappenbach/cajeta-http) library
  (HTTP/WS/SSE over the `cajeta.io.net` transport). primavera owns the *policy*, not
  the HTTP engine or the executor (inherited from `cajeta.io.net`).
- **Stereotypes** — `@Repository`, `@Service` (named roles over `@Component`).
- **Deployment profiles** — `@Profile("prod"|"test"|...)`.
- **Test harness** — `@TestComponent` overrides + request-scope seeding over the
  core `@Inject` override seam. See [`Testing.md`](Testing.md).

For the `@Factory` design rationale (the `@Bean` rejection, the five-pillar
defense of canonical construction), see [`Factory.md`](Factory.md) — but the
normative `@Factory` spec is the stdlib `AspectModel.md`.
