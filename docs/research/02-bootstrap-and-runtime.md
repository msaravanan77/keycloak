# 2. Bootstrap & Request Lifecycle

This chapter answers: *what actually runs when you execute `kc.sh start`,
and what happens to the very first byte of an incoming HTTP request?*

## 2.1 Process entry point

```
bin/kc.sh
  └─ exec java ... -cp quarkus-run.jar io.quarkus.bootstrap.runner.QuarkusEntryPoint   (Quarkus fast-jar bootstrap)
       └─ resolves @QuarkusMain → org.keycloak.quarkus.runtime.KeycloakMain.main(String[])
```

`quarkus/dist/src/main/content/bin/kc.sh` execs into Quarkus's fast-jar
runner. That resolves the application's `@QuarkusMain` class:

**`org.keycloak.quarkus.runtime.KeycloakMain`**
(`quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/KeycloakMain.java:60`)

```java
@QuarkusMain(name = "keycloak")
public class KeycloakMain implements QuarkusApplication {
    public static void main(String[] args) { ... }
    ...
}
```

`main()` builds a Picocli CLI (`org.keycloak.quarkus.runtime.cli.Picocli`)
and calls `picocli.parseAndRun(cliArgs)`, which dispatches to picocli
subcommands under `quarkus/runtime/.../cli/command/` — `Main`, `Start`
(`extends AbstractAutoBuildCommand`), `Build`, etc. The `start` command
eventually calls `KeycloakMain.start(...)`, which calls
`io.quarkus.runtime.Quarkus.run(KeycloakMain.class, ...)` — this is where
control passes into standard Quarkus bootstrap: CDI/Arc container
initialization, running all `@BuildStep`-generated startup code, firing
`StartupEvent`. `KeycloakMain implements QuarkusApplication`, so once Quarkus
itself is up, its `run(String... args)` callback fires — it fetches
`QuarkusKeycloakApplication` / `QuarkusKeycloakSessionFactory` from the Arc
CDI container and calls `Quarkus.waitForExit()` to block the main thread for
the life of the server.

## 2.2 Build-time vs. runtime: the Quarkus extension

Keycloak ships as its own Quarkus **extension**, split (like every Quarkus
extension) into a `deployment` artifact (runs only during image
build/augmentation, in a throwaway JVM) and a `runtime` artifact (compiled
into the actual server).

**`org.keycloak.quarkus.deployment.KeycloakProcessor`**
(`quarkus/deployment/src/main/java/org/keycloak/quarkus/deployment/KeycloakProcessor.java`,
~1180 lines, every method a `@BuildStep`) is where the interesting
build-time decisions happen:

- `getFeature()` — registers the `keycloak` Quarkus feature.
- `initConfig()` (`@Record(STATIC_INIT)`) — calls `Config.init(new MicroProfileConfigProvider())` so `org.keycloak.Config` is backed by MicroProfile Config from the very first static-init phase.
- `filterAllRequests()` / `filterAllManagementRequests()` (`@Record(RUNTIME_INIT)`) — install a "reject non-normalized path" filter and a "misdirected request" filter directly on the Vert.x router.
- `configureResteasy()` — registers RESTEasy Reactive customizers (see §2.3).
- `configureKeycloakSessionFactory()` (`@Record(STATIC_INIT)`, line ~665) — **the most important build step**: discovers every enabled `Provider`/`ProviderFactory` implementation via `org.keycloak.provider.ProviderManager` (Keycloak's own SPI scanner reading `META-INF/services/org.keycloak.provider.Spi` and factory service files), and captures the resulting `Map<Spi, List<Class<? extends ProviderFactory>>>` as a build-time artifact — only the **class list** is baked in; actual instantiation is deferred to runtime so config (`keycloak.conf`, env vars) can still affect which factories run and how. Non-built-in (user-deployed) providers are loaded at server startup instead, not at build time.

At `STATIC_INIT`, the corresponding `KeycloakRecorder`
(`quarkus/runtime/.../KeycloakRecorder.java`) — Quarkus's bytecode-recording
mechanism — replays these build-time decisions into the actual running
process, e.g. `KeycloakRecorder.createSessionFactory(...)` builds the real
`QuarkusKeycloakSessionFactory`.

## 2.3 Anatomy of one HTTP request

Quarkus's HTTP layer here is **Vert.x** (not Undertow/servlet). The request
path:

```
TCP socket
 → Vert.x HttpServer
   → Vert.x router  (path-normalization + misdirected-request filters, installed by KeycloakProcessor)
     → RESTEasy Reactive routing (generated at build time)
       → KeycloakHandlerChainCustomizer-installed handlers:
           - FormBodyHandler          (parses form bodies before POST/PUT/PATCH invocation)
           - TransactionalSessionHandler  (alternateInvocationHandler — see below)
           - SetResponseContentTypeHandler
       → JAX-RS resource method   (e.g. RealmsResource, AuthorizationEndpoint, TokenEndpoint, AdminRoot ...)
       ← KeycloakSecurityHeadersFilter (@Provider ContainerResponseFilter — adds security headers)
       ← DefaultCors / DefaultCorsFactory (CORS handling)
       ← CloseSessionFilter (@Provider @PreMatching @Priority(1) — closes/commits the KeycloakSession)
```

- **`org.keycloak.quarkus.runtime.integration.resteasy.KeycloakHandlerChainCustomizer`**
  (`implements HandlerChainCustomizer`) is how Keycloak injects itself into
  RESTEasy Reactive's per-request handler chain. Its
  `alternateInvocationHandler` swaps in
  **`TransactionalSessionHandler`** (same package,
  `extends InvocationHandler`): on invocation it fetches the CDI-scoped
  `KeycloakSession`, calls `KeycloakSessionUtil.setKeycloakSession(session)`
  (a thread-local), and — if running on a blocking worker thread with no
  active transaction — calls `beginTransaction(session)` *before* the actual
  JAX-RS method runs.
- **Root JAX-RS resource**: `org.keycloak.services.resources.RealmsResource`
  (`services/src/main/java/org/keycloak/services/resources/RealmsResource.java:69`),
  annotated `@Path("/realms")`, registered through
  `QuarkusKeycloakApplication` (`@ApplicationPath("/")`, extends
  `org.keycloak.services.resources.KeycloakApplication`). Everything
  realm-scoped — login, protocol endpoints, well-known config, account
  console — hangs off this resource as sub-resource locators keyed by
  `{realm}`. The **admin** REST tree is a sibling root,
  `org.keycloak.services.resources.admin.AdminRoot` (`@Path("/admin")`) —
  see chapter 8.
- **Response side**: `KeycloakSecurityHeadersFilter` adds standard security
  headers; CORS is handled by `DefaultCors`/`DefaultCorsFactory`;
  **`CloseSessionFilter`**
  (`quarkus/runtime/.../integration/jaxrs/CloseSessionFilter.java`,
  `@Provider @PreMatching @Priority(1)`) commits/rolls back the
  `KeycloakSession`'s transaction and closes it at the end of the request
  (with special handling for streaming/`StreamingOutput` entities, closing
  only once the stream itself finishes). A CDI disposer method in
  `KeycloakBeanProducer.dispose(@Disposes KeycloakSession session)` is a
  safety net that closes the session if `CloseSessionFilter` somehow didn't
  run.

## 2.4 `KeycloakSession`: the per-request unit of work

`KeycloakSession` (`server-spi/src/main/java/org/keycloak/models/KeycloakSession.java`)
is **not** an HTTP session — it is a per-request handle bundling:

- a `KeycloakTransactionManager` (the transaction boundary for this request),
- a `KeycloakContext` (realm/client/URI context for this request),
- and a provider registry: `getProvider(Class<T>)` /
  `getProvider(Class<T>, String id)`, plus convenience accessors used
  everywhere in the codebase — `session.realms()`, `session.users()`,
  `session.clients()`, `session.sessions()`, `session.authenticationSessions()`,
  and so on (each backed by a corresponding `*Provider` SPI — see chapter 7).

Creation: `KeycloakBeanProducer.getKeycloakSession()`
(`@RequestScoped`, `quarkus/runtime/.../integration/cdi/KeycloakBeanProducer.java`)
calls `factory.create()` on the injected `QuarkusKeycloakSessionFactory` —
**lazily**, on first use, which matters because it lets code run safely on
the Vert.x event loop before a session is actually needed.
`QuarkusKeycloakSessionFactory.create()`
(`quarkus/runtime/.../integration/QuarkusKeycloakSessionFactory.java:78`)
returns `new QuarkusKeycloakSession(this)`.

`QuarkusKeycloakSessionFactory` itself
(`extends org.keycloak.services.DefaultKeycloakSessionFactory`) is
constructed once at `STATIC_INIT`: its constructor instantiates every
build-time-discovered `ProviderFactory` class by reflection
(`lookupProviderFactory`), calls `factory.init(Config.scope(spi, factoryId))`
on each, and populates the internal `factoriesMap` used by every later
`getProvider()` lookup.

## 2.5 Configuration

CLI parsing goes through `Picocli`
(`quarkus/runtime/.../cli/Picocli.java`, using `picocli.CommandLine`),
commands under `cli/command/`. Actual configuration values are resolved
through a layered SmallRye/MicroProfile `Config`, with Keycloak-specific
sources under `quarkus/runtime/.../configuration/`:

- `KeycloakPropertiesConfigSource` / `QuarkusPropertiesConfigSource` — reads `conf/keycloak.conf`.
- `KcEnvConfigSource` — `KC_*` environment variables.
- `ConfigArgsConfigSource` — `--option=value` CLI arguments.
- `PersistedConfigSource` — build-time-persisted properties baked into an augmented server image (cleared on `KeycloakMain.reset`).

`Configuration.java` is the facade most runtime code calls
(`Configuration.getOptionalKcValue(...)`), and `MicroProfileConfigProvider`
bridges this whole stack to `org.keycloak.Config`, the config API used
pervasively by SPI provider factories (`factory.init(Config.Scope config)`).

## 2.6 First-boot realm/DB initialization

`org.keycloak.services.resources.KeycloakApplication.startup()`
(`services/.../resources/KeycloakApplication.java:76`) is where the very
first realm data gets created:

1. Initializes crypto (`CryptoIntegration.init`) and the temp directory.
2. Builds the `KeycloakSessionFactory`, then calls private `runBootstrap(sessionFactory)`:
   - `sessionFactory.init()` — publishes lifecycle events (e.g. `postInit`) to every provider factory.
   - Opens a `KeycloakSession` via `KeycloakModelUtils.runJobInTransactionWithResult`.
   - Uses `DBLockManager`/`DBLockProvider` to take a **cluster-wide DB lock** (`Namespace.KEYCLOAK_BOOT`) so that in a multi-node deployment, only one node performs first-boot bootstrap/migration.
   - Calls `bootstrap(session)`, which returns an `ExportImportManager`
     (`server-spi-private/.../storage/ExportImportManager.java`, impl in
     `services/.../exportimport/ExportImportManager.java`) — this drives
     master-realm creation and any configured export/import (e.g.
     `KC_IMPORT`/`--import-realm`).
3. `QuarkusKeycloakApplication.createTemporaryAdmin(session)`
   (`quarkus/runtime/.../integration/jaxrs/QuarkusKeycloakApplication.java:108`)
   uses `org.keycloak.services.managers.ApplianceBootstrap` to create the
   temporary master-realm admin account from `KEYCLOAK_ADMIN` /
   `KC_BOOTSTRAP_ADMIN_*` options — this is literally where the account you
   log into the Admin Console with the first time comes from.
4. `sessionFactory.publish(new PostMigrationEvent(...))` and
   `setBootstrapCompleted()` fire, unblocking normal traffic.

In `QuarkusKeycloakApplication` this whole sequence is wired to Quarkus's
`@Observes StartupEvent`, optionally run asynchronously via a
`ManagedExecutor` (configurable; synchronous in dev/test modes) so the
server can report "ready" without necessarily blocking on bootstrap.

## 2.7 Why this matters when reading the rest of the codebase

Two habits fall directly out of this chapter and recur everywhere else in
this knowledge base:

- **"Where does X come from?" is almost always answered by "a `Provider`
  looked up on `KeycloakSession`."** If you're reading unfamiliar code and
  see `session.getProvider(Foo.class)` or `session.foos()`, that's your cue
  to go find `Foo`'s `ProviderFactory` implementations exactly the way
  `configureKeycloakSessionFactory` discovers them.
- **Every request is transactional by construction.** `TransactionalSessionHandler`
  begins a transaction before your JAX-RS method runs, and `CloseSessionFilter`
  commits/rolls it back after. Code inside a resource method essentially
  never manages transactions manually.

Continue to [03-authentication-engine.md](03-authentication-engine.md).
