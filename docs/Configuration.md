# Configuration & value injection

primavera's typed configuration system: `@Value` and `@Config` bind external
configuration into components at **compile time** (generated binders, no
reflection). This is the deep-dive for `primavera-spec.md` §4.

## Why this is one mechanism for three targets

Configuration looks different across Cajeta's targets, but one mechanism serves all
three — which is the point:

- **Enterprise** — externalized config (endpoints, pool sizes, feature flags,
  secrets) that varies per deployment without recompiling. Spring `@Value` /
  `@ConfigurationProperties`, Micronaut `@Property`.
- **ML / DS / physics** — per-run **experiment config**: hyperparameters, model /
  dataset / solver selection, precision, device. This is config *composition* — the
  problem Hydra exists to solve (§10).
- **Embedded** — config can be **frozen at compile time** into constants (no
  runtime file read, DCE-friendly), when the deployment is fixed (§6).

The same `@Config` type, the same binding rules — only the *sources* and the
compile-time-vs-runtime split differ.

## 1. The two surfaces

### `@Value` — a single value

```cajeta
@Component class Server {
    @Value("server.port") int32 port;                       // required
    @Value("server.host", default="0.0.0.0") String host;   // with default
    @Value("server.tags") String[] tags;                    // list
}
```

`@Value` injects one resolved value into a field or constructor/factory parameter
(it composes with `@Inject` constructors — a parameter may be `@Value`-bound while
its siblings are `@Inject`-resolved). A `@Value` with no `default=` and no resolved
source value is a **startup failure** (or compile error in frozen mode, §6).

### `@Config` — a typed group

```cajeta
@Config(prefix="db") class DbConfig {
    String   url;                                  // db.url        (required)
    int32    poolSize    = 8;                       // db.pool-size  (default in the type)
    Duration idleTimeout = Duration.ofSeconds(90);  // db.idle-timeout
    TlsMode  tls         = TlsMode.PREFERRED;        // enum
    Replica[] replicas;                              // db.replicas[] (nested list)
}
@Config(prefix="db.replica") class Replica { String host; int16 port = 5432; }

@Component class Repo { @Inject DbConfig cfg; }     // injected like any component
```

`@Config` binds a whole subtree under `prefix` into a typed object — the preferred
shape for anything beyond a single value: it is type-checked, validated as a unit
(§7), nests, and is injected by `@Inject` like any `@Component`. Field defaults are
the source of truth for "optional"; a field with no default is required.

## 2. Sources & precedence

Resolved config is a merge of layered sources, **highest precedence first**:

1. **CLI args** — `--db.pool-size=16`, `--server.port=8443`.
2. **Environment** — `DB_POOL_SIZE`, `SERVER_PORT` (see name mapping below).
3. **Profile file** — `application-<profile>.toml` for the active `@Profile`.
4. **Base file** — `application.toml`.
5. **Type defaults** — field initializers in `@Config` types / `@Value(default=)`.

A higher layer overrides a lower one *key by key* (not whole-file). The canonical
format is **TOML** (typed, comment-friendly); additional formats and remote sources
plug in via the `ConfigSource` seam (§9).

**Name mapping (relaxed binding).** Canonical keys are dot-path **kebab-case**
(`db.pool-size`); fields are camelCase (`poolSize`); the binder maps between them.
Environment variables uppercase with `_` separators (`DB_POOL_SIZE`). A list key is
`db.replicas` (TOML array) or `DB_REPLICAS_0_HOST` (env, indexed).

```toml
# application.toml
[server]
port = 8080
host = "0.0.0.0"

[db]
url = "postgres://localhost/app"
pool-size = 16
replicas = [ { host = "r1", port = 5432 }, { host = "r2" } ]
```

## 3. Binding & coercion

The binder coerces source strings/values into typed fields:

| Target | Accepts |
|---|---|
| `int*`/`float*`/`boolean` | numeric / `true`/`false` literals; range-checked |
| `String` | as-is |
| `Duration` | `"90s"`, `"5m"`, `"1h30m"`, ISO-8601 |
| `enum` | case-insensitive variant name |
| `T[]` / `List<T>` | TOML array, or comma-split env/arg string |
| `Map<String,V>` | TOML table / `prefix.*` keys |
| nested `@Config` type | a subtree (recursive bind) |

Coercion failure (`pool-size = "lots"`), a missing required key, or an unknown key
under a `@JsonStrict`-style strict `@Config` is a **fail-fast** error (§7).

## 4. Compile-time vs runtime — the AoT contract

This is the honest core of an AoT config system. Config *values* are inherently
runtime (varying per deployment is the whole point), but everything *around* them is
compile-time:

- **Binders are generated at compile time.** For each `@Config` type, the compiler
  knows the full field set and emits a direct binder — read `db.url`, coerce to
  `String`, read `db.pool-size`, coerce to `int32`, apply the default, validate,
  construct the owned object. **No reflection, no runtime field walk.** (Mechanically
  this is a synthesized `@Factory` for the config type; see `Factory.md`.)
- **Type & coercion *shape* are checked at compile time** — the target types are
  known, so "this key coerces to `int32`" is settled before the binary is built.
- **Presence & values are checked at runtime startup** — the binder runs once at
  boot, fails fast on a missing-required / bad-coercion across the merged sources,
  and prints which key + which source.
- **Key typos are caught structurally** where the key set is closed: a `@Value("srever.port")`
  referencing a key no `@Config` type or known schema declares can be a compile
  warning/error (the declared-key set is statically known).

So "compile-time config" = generated binder + compile-time type/coercion checking +
runtime fail-fast on values. That is the most an AoT system can honestly promise,
and it is strictly more than Spring's all-runtime reflection model.

## 5. Frozen (compile-time-baked) config — embedded

When a deployment is fixed (an embedded/robotics image), config can be **frozen**:
the build reads a checked-in config file and bakes the resolved values as
compile-time **constants**, with *no runtime source reading and no config machinery
linked* (DCE drops the source readers). Selected per build:

```
cajeta build --config-mode=frozen --config=device.toml
```

In frozen mode, a missing-required key is a **compile error** (not a startup
failure), and the entire sources/precedence/parse path is dead-code-eliminated —
only the constant values remain. Runtime mode (the default for servers / ML) keeps
the layered sources. The `@Config` types are identical across modes.

## 6. Validation & errors

`@Config` fields carry the same constraints as request bodies (`primavera-spec.md`
§5), checked when the binder runs:

```cajeta
@Config(prefix="db") class DbConfig {
    @NotBlank String url;
    @Min(1) @Max(256) int32 poolSize = 8;
}
```

Startup validation aggregates **all** violations into one report (not first-fail)
and aborts boot — a 12-factor fail-fast. Each error names the key, the offending
value, the source it came from, and the constraint. In frozen mode these are compile
errors.

## 7. Interpolation, references & secrets

Values may reference other values, the environment, or a secret store:

```toml
[db]
host     = "db.internal"
url      = "postgres://${db.host}/app"        # reference another key
log-dir  = "${env:LOG_DIR:/var/log}"          # env var with a default
password = "${secret:db-password}"            # resolved via a SecretSource
```

- `${key}` — another config key (cycles are a startup error).
- `${env:VAR}` / `${env:VAR:default}` — environment.
- `${secret:name}` — resolved through the **`SecretSource` seam** (env / Vault / KMS;
  ship env + one provider). Secret-typed values are flagged so they are **never
  logged** and excluded from the effective-config dump (§11).

## 8. Conditional wiring (`@Requires`)

Config drives **build-time conditional graph inclusion** — the Micronaut `@Requires`
analog that closes most of the enterprise "flexibility" gap without a runtime
container:

```cajeta
@Component @Requires(property="cache.enabled", value="true")
class RedisCache implements Cache { ... }

@Component @Requires(missing=RedisCache.class)
class InMemoryCache implements Cache { ... }     // fallback when Redis isn't wired
```

A component is included in the graph only when its `@Requires` predicate holds
(property present/equal, profile active, class present/missing, platform). The
decision stays in the analyzable compile-time graph, not in imperative factory code.

## 9. Pluggable sources — the `ConfigSource` seam

```cajeta
interface ConfigSource {
    String name();
    int32  priority();                       // slots into the precedence ladder (§2)
    Optional<String> get(String key);
    Iterable<String> keys();                 // for strict-unknown-key checks + dumps
}
```

Built-in sources (args, env, files) implement it; a config-server / Consul / Vault
source is an ecosystem library that registers at a chosen priority. The merge engine
is source-agnostic.

## 10. ML / DS experiment config (the differentiator)

The same `@Config` mechanism makes experiment configuration first-class — the Hydra
use case, but typed and compile-checked:

- **Config groups + a defaults list.** Select a variant per axis (model, dataset,
  optimizer) from a group of files, composed into one resolved tree:

  ```toml
  # experiment.toml
  defaults = [ { model = "resnet50" }, { optimizer = "adamw" }, { dataset = "imagenet" } ]
  seed = 1337
  ```
  with `conf/model/resnet50.toml`, `conf/optimizer/adamw.toml`, etc. selected and merged.

- **CLI override & sweep.** `--model=vit_b16 --optimizer.lr=3e-4` overrides any leaf
  (the CLI layer, §2). A typed `@Config ExperimentConfig` binds the result, so a
  hyperparameter typo or out-of-range value fails before the GPU spins up.

- **Reproducibility.** The **effective-config dump** (§11) records the exact merged
  config a run used — the experiment's provenance — alongside the seed.

This unifies "what Spring config is for" and "what Hydra is for" under one typed,
no-reflection mechanism, which neither the JVM frameworks nor Hydra do.

## 11. Lifecycle, ownership & the effective-config dump

- **Owned, immutable snapshot.** A bound `@Config` object is **owned** and immutable
  after binding — deterministically dropped with its scope, no GC, no surprise
  mutation. Most config is singleton-scoped.
- **Reload (opt-in).** A `@Config(reload=true)` type exposes a `ConfigWatcher` that
  rebinds a **new** immutable snapshot on source change and swaps it atomically;
  consumers read through a handle. Off by default (immutability is the safe norm);
  useful for long-running servers and ML dashboards.
- **Effective-config dump.** At startup (and on demand via a management endpoint,
  `primavera-spec.md` §12) primavera can emit the fully-resolved config with each
  value's winning source — `db.pool-size = 16 (from: env DB_POOL_SIZE)` — secrets
  redacted. The single most useful ops + reproducibility artifact.

## 12. Worked examples

**Enterprise service:**
```cajeta
@Config(prefix="db") class DbConfig { @NotBlank String url; @Min(1) int32 poolSize = 8; }
@Component class OrderRepo { @Inject DbConfig db; }
// --db.pool-size=32 (CLI) overrides DB_POOL_SIZE (env) overrides application-prod.toml.
```

**ML experiment:**
```cajeta
@Config(prefix="train") class TrainConfig {
    @Min(1) int32 epochs = 90;
    @Min(0.0) float64 lr  = 0.1;
    Device device = Device.GPU0;
    @Inject ModelConfig model;          // selected via the defaults list / --model=
}
// cajeta run -- --train.lr=3e-4 --model=vit_b16   →  typed, validated, dumped for the run log.
```

## 13. Open questions

- **Format** — TOML primary (typed, clean). Also accept YAML, or hold the line on
  TOML + the `ConfigSource` seam for the rest? Lean: TOML-only built-in.
- **Strict-unknown-key default** — reject unknown keys under a `@Config` by default
  (catches typos) or warn? Lean: warn by default, `@Config(strict=true)` to reject.
- **Frozen-mode partial** — allow *some* keys frozen and others runtime in one build,
  or all-or-nothing? Lean: all-or-nothing per build for simplicity.
- **Reload granularity** — per-`@Config`-type swap (specified) vs finer field-level
  reactivity. Lean: type-level snapshot swap.
- **Compile-time key validation strength** — warning vs hard error for an undeclared
  `@Value` key. Lean: error in frozen mode, warning in runtime mode.
