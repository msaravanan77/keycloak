# 1. The Big Picture

## 1.1 What Keycloak actually is, architecturally

Keycloak is a single Java server process (a Quarkus application) that
implements **three cooperating roles** behind one set of REST/HTTP endpoints:

1. **An OpenID Connect Provider / SAML 2.0 IdP** — it authenticates end
   users (via its own login pages) and issues tokens/assertions to
   applications ("clients").
2. **An identity broker** — it can also delegate the actual authentication
   decision to *other* IdPs (Google, GitHub, another Keycloak, any OIDC/SAML
   IdP) and then locally represent the result as a Keycloak session.
3. **A user directory with federation** — user records can live in
   Keycloak's own database, or be sourced live from LDAP/Kerberos/a custom
   store, cached locally.

All three roles share one authentication **flow engine**, one **domain
model** (Realm/Client/User/Session), one **persistence + cache stack**, and
one **admin/REST management plane**. That sharing is the single most
important architectural fact about this codebase: OIDC login, SAML login,
brokered login, and LDAP-backed login are not four different code paths —
they are four different *inputs* into the same `AuthenticationProcessor`
engine, which then hands off to a pluggable `LoginProtocol` (OIDC or SAML)
for the final response. See chapters 3–5.

## 1.2 Module map (Java-relevant modules only)

Keycloak is a large multi-module Maven reactor (`pom.xml` at repo root,
`<version>999.0.0-SNAPSHOT</version>`, `<packaging>pom</packaging>`). The
modules that matter for understanding the running server:

| Module | Role |
|---|---|
| `quarkus/runtime` | The actual runtime glue: `KeycloakMain` (process entry point), CLI (`Picocli`), config sources, CDI producers, the RESTEasy handler chain, `QuarkusKeycloakSessionFactory`. |
| `quarkus/deployment` | Quarkus **build-time** extension (`KeycloakProcessor`, all `@BuildStep`s) — runs during `kc.sh build`/augmentation, not at request time. Decides which SPI provider classes get baked into the server image. |
| `quarkus/dist` | Packaging: `kc.sh`/`kc.bat`, `conf/keycloak.conf` templates, the distributable zip/container layout. |
| `quarkus/server` | Assembles the actual server artifact from the above. |
| `server-spi` | The **public SPI contracts**: `KeycloakSession`, `RealmModel`, `ClientModel`, `UserModel`, `Provider`/`ProviderFactory`, `AuthenticationSessionModel`, etc. This is the "interfaces" module — almost no logic. |
| `server-spi-private` | Internal-but-shared SPI contracts not meant for third-party extension: `Authenticator`, `RequiredActionProvider`, `LoginProtocol`, `ClusterProvider`, event interfaces, broker `IdentityProvider` SPI, authorization-services model/store interfaces. |
| `services` | **The bulk of the runtime logic.** All JAX-RS resources (`RealmsResource`, `LoginActionsService`, admin resources, protocol endpoints), the `AuthenticationProcessor` flow engine, built-in `Authenticator`/`RequiredActionProvider` implementations, identity-broker implementations (OIDC/SAML/social), the OIDC/SAML protocol implementations, authorization-services runtime, event dispatch, brute-force protection. |
| `model/jpa` | Hibernate/JPA-backed implementation of the `server-spi` model interfaces (`JpaRealmProvider`, entities, Liquibase changelogs). This is where a Realm/User/Client actually gets read from/written to a relational database. |
| `model/infinispan` | Infinispan-backed **cache decorators** over the JPA providers (`RealmCacheSession`, `UserCacheSession`) plus **session storage** (user sessions, auth sessions, login-failure/brute-force counters) and cross-node clustering (`ClusterProvider`, JGroups). |
| `model/storage`, `model/storage-private`, `model/storage-services` | The **User Storage SPI** — the abstraction LDAP/Kerberos/custom federation plug into (`UserStorageProvider`), plus the cache SPI shape used by `model/infinispan`. |
| `federation/ldap`, `federation/kerberos` | Concrete federation providers: LDAP user storage + mappers, Kerberos/SPNEGO authentication and ticket-based user import. |
| `saml-core`, `saml-core-api` | Low-level SAML 2.0 protocol library: XML (un)marshalling, XML-DSig signing/verification, assertion/response builders. Used by `services/.../protocol/saml`. |
| `authz/policy/common` | Built-in `PolicyProvider` implementations (role/user/client/group/time/regex/js/aggregated) for Authorization Services. |
| `crypto` | Pluggable crypto backends (default vs. FIPS), signature/cipher providers. |
| `core` | Shared representation classes (`AccessToken`, `IDToken`, adapter config classes like `PolicyEnforcerConfig`) used by both server and client-side adapters — no server logic. |
| `operator` | A **separate Kubernetes Operator** (Java, Quarkus + Java Operator SDK / Fabric8), for declaratively deploying/upgrading Keycloak and importing realms on K8s. Architecturally independent of the runtime auth server — see chapter 8. |
| `js/apps/admin-ui`, `js/apps/account-ui` | The Admin Console and Account Console single-page apps (TypeScript/React). Out of scope here except for orientation — they are HTTP clients of the Java REST APIs described in chapter 8. |
| `adapters/`, `authz/` (client side) | Client-side integration libraries (framework adapters, policy enforcer contracts) for applications that want to *use* Keycloak. Largely out of scope for "how the server works." |

## 1.3 The one-paragraph request story

An HTTP request hits Quarkus's Vert.x-based HTTP layer, flows through a
small chain of Keycloak-installed filters, and lands on a JAX-RS resource
rooted at `RealmsResource` (`@Path("/realms")`, in `services`). Every such
request gets a fresh `KeycloakSession` (a **unit-of-work object** —
transaction + provider registry, not an HTTP session) created from the
`QuarkusKeycloakSessionFactory`. If the request is a login-related one, it's
handled by `AuthorizationEndpoint`/`LoginActionsService`, which drive the
`AuthenticationProcessor` flow engine through a realm-configured sequence of
`Authenticator`s (password form, OTP, WebAuthn, Kerberos/SPNEGO, "redirect to
external IdP," ...). Once the flow completes, control passes to a
`LoginProtocol` implementation (OIDC or SAML) which mints/signs tokens or
assertions using the realm's keys and redirects the browser back to the
client application. Every domain object touched along the way (`RealmModel`,
`UserModel`, `UserSessionModel`, ...) is fetched through `KeycloakSession`'s
provider accessors, which resolve — by default — to an **Infinispan cache
decorator wrapping a JPA provider wrapping a relational database**, with
cross-node cache invalidation and (for sessions) Infinispan-clustered or
DB-persistent session storage.

## 1.4 Why the SPI/Provider pattern is everywhere

Almost every noun in Keycloak (`RealmProvider`, `UserProvider`,
`Authenticator`, `IdentityProvider`, `EventListenerProvider`,
`PolicyProvider`, `SignatureProvider`, `CredentialProvider`, `LoginProtocol`,
`UserStorageProvider`...) is a `Provider` produced by a `ProviderFactory`,
discovered through Keycloak's own SPI mechanism (`org.keycloak.provider.Spi`
+ `ProviderManager`, effectively a curated `ServiceLoader`), and looked up
per-request via `session.getProvider(SomeInterface.class[, id])`. This is
*the* extension point pattern in the codebase: to add a new authentication
step, a new identity broker type, a new event sink, a new policy type, or a
new signature algorithm, you implement a `Provider`+`ProviderFactory` pair
and register it — you never modify the flow engine, the protocol endpoints,
or the REST layer itself. Understanding this pattern once means you can read
almost any subsystem in this codebase by asking "what SPI is this, what's
the factory, what are the built-in implementations."

Continue to [02-bootstrap-and-runtime.md](02-bootstrap-and-runtime.md) for
exactly how this all starts up and handles its first request.
