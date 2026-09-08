# 7. Domain Model, Persistence & Caching

Every chapter so far has been reading/writing `RealmModel`, `UserModel`,
`ClientModel`, `UserSessionModel`, etc. through `KeycloakSession`. This
chapter is what's actually behind those interfaces.

## 7.1 The domain model interfaces (`server-spi/.../models/`)

- **`RealmModel`** (`extends RoleContainerModel`) — a tenant/security realm. Owns clients, roles, groups, identity providers, client scopes, authentication flows. Declares `ProviderEvent` types fired via `KeycloakSessionFactory` (`RealmCreationEvent`, `RealmPostCreateEvent`, `RealmRemovedEvent`, `IdentityProviderUpdatedEvent`, ...) — this is how cache invalidation (§7.4) and other cross-cutting concerns learn about model changes.
- **`ClientModel`** (`extends ClientScopeModel, RoleContainerModel, ProtocolMapperContainerModel, ScopeContainerModel, Model`) — an OAuth/OIDC/SAML client application registered in a realm: protocol mappers, roles, redirect URIs, secrets.
- **`UserModel`** (`extends RoleMapperModel, Model`) — an account: role mappings (direct + via group membership), attributes, required actions, federation link (chapter 4).
- **`RoleModel`**, **`GroupModel`** (`extends RoleMapperModel, Model`) — roles can be realm-level or client-level (both `RealmModel` and `ClientModel` implement `RoleContainerModel`); groups form a parent/subgroup hierarchy (`getParent()`, `getSubGroupsStream()`) and carry their own role mappings, so users inherit roles transitively through group membership.
- **`UserSessionModel`** — a browser/SSO login session: `getUser()` → `UserModel`, `getAuthenticatedClientSessions()` → `Map<clientUuid, AuthenticatedClientSessionModel>`, plus notes/state.
- **`AuthenticatedClientSessionModel`** (`extends CommonClientSessionModel`) — one per (user session × client): tracks refresh tokens (`getRefreshToken(reuseId)`), back-reference `getUserSession()`, `detachFromUserSession()`.

**Relationship chain**: `RealmModel` 1—\* `ClientModel`/`RoleModel`/`GroupModel`;
`UserModel` \*—\* `GroupModel`/`RoleModel`; `UserSessionModel` 1—\*
`AuthenticatedClientSessionModel` (one per client touched), each tied to one
`UserModel` and one `RealmModel`.

## 7.2 The JPA persistence layer (`model/jpa`)

Package `org.keycloak.models.jpa`. ORM: **Hibernate**, via Jakarta
Persistence annotations (plus some Hibernate-specific ones, e.g.
`@Nationalized` on `RealmEntity`).

- **Providers/factories**: `JpaRealmProvider(+Factory)` (`PROVIDER_ID="jpa"`), `JpaClientProviderFactory`, `JpaGroupProviderFactory`, `JpaRoleProviderFactory`, `JpaUserProvider(+Factory)`, `JpaUserCredentialStore`, `JpaIdentityProviderStorageProvider`.
- **SPI↔entity adapters**: `RealmAdapter`, `ClientAdapter`, `UserAdapter`, `GroupAdapter`, `RoleAdapter`, `ClientScopeAdapter` — implement the `server-spi` interfaces from §7.1, each wrapping a JPA entity + `EntityManager`.
- **Entities** (`model/jpa/src/main/java/org/keycloak/models/jpa/entities/`): `RealmEntity`, `RealmAttributeEntity`, `ClientEntity`, `ClientAttributeEntity`, `UserEntity`, `UserAttributeEntity`, `RoleEntity`, `CompositeRoleEntity`, `GroupEntity`, `GroupRoleMappingEntity`, `UserRoleMappingEntity`, `UserGroupMembershipEntity`, `CredentialEntity`, `FederatedIdentityEntity`, `ComponentEntity`, plus organization/authorization-services entities.
- **DB connectivity**: `org.keycloak.connections.jpa.JpaConnectionProvider` supplies the `EntityManager` these providers use.

### No alternate "map storage" backend currently

Earlier Keycloak versions experimented with a non-JPA "map storage" SPI —
it does **not** exist in this checkout (`model/map` is absent). Only
`model/jpa` (relational) and `model/infinispan` (cache/session) exist under
`model/`. `model/storage*` holds the **User Storage SPI** (federation,
chapter 4) and cache SPI shapes, not an alternate core-model persistence
backend.

## 7.3 `KeycloakSession` as the unit-of-work facade

`server-spi/.../models/KeycloakSession.java` exposes provider accessors as
the per-request entry points into all of this:

```java
RealmProvider realms();
ClientProvider clients();
ClientScopeProvider clientScopes();
GroupProvider groups();
RoleProvider roles();
UserProvider users();
UserSessionProvider sessions();
UserLoginFailureProvider loginFailures();
AuthenticationSessionProvider authenticationSessions();
SingleUseObjectProvider singleUseObjects();
RevokedTokenProvider revokedTokens();
IdentityProviderStorageProvider identityProviders();
<T extends Provider> T getProvider(Class<T> clazz);
<T extends Provider> T getProvider(Class<T> clazz, String id);
```

At runtime, `session.realms()` / `session.users()` resolve — through the
SPI/`ProviderFactory` registry described in chapter 2 — to whichever
provider is configured. **By default that's the Infinispan cache decorator,
wrapping the JPA provider**, i.e.:

```
request code  →  session.realms()  →  RealmCacheSession (cache-through)  →  JpaRealmProvider  →  Hibernate  →  DB
```

## 7.4 Caching layer (Infinispan-backed, L1-per-node with invalidation)

Cache SPI interfaces:

- `model/storage/.../models/cache/`: `UserCache`, `CachedUserModel`, `CachedObject`, `OnUserCache`.
- `model/storage-private/.../models/cache/`: `CacheRealmProvider(+Factory)`, `CacheRealmProviderSpi`, `CacheUserProviderSpi`, `UserCacheProviderFactory`, `CachedRealmModel`, `CachePublicKeyProvider(Factory)`, `CacheCrlProvider(Factory)`.

Implementation, `model/infinispan/.../models/cache/infinispan/`:

- **`RealmCacheSession`** (`implements CacheRealmProvider`) — decorator over the JPA `RealmProvider`: reads check the local Infinispan cache first, only falling through to JPA on a miss, then populate the cache. Returned adapters (`RealmAdapter`, `ClientAdapter`, `RoleAdapter`, `GroupAdapter` — cache-flavored, implementing `CachedRealmModel` etc.) are distinct from the JPA-flavored adapters in §7.2, though same-named.
- **`UserCacheSession`** — the same pattern over `UserProvider` (this is also what fronts LDAP-federated user lookups, chapter 4).
- **`RealmCacheManager`** (`extends CacheManager`) / **`UserCacheManager`** — own the actual Infinispan `Cache` and a revision/versioning scheme (`UpdateCounter`) to detect staleness.
- Factories: `InfinispanCacheRealmProviderFactory`, `InfinispanUserCacheProviderFactory`.
- **Invalidation events** (`.../cache/infinispan/events/`): `RealmUpdatedEvent`, `RealmRemovedEvent`, `ClientAddedEvent`/`UpdatedEvent`/`RemovedEvent`, `RoleAddedEvent`/`UpdatedEvent`/`RemovedEvent`, `GroupAddedEvent`/`UpdatedEvent`/`RemovedEvent`/`MovedEvent`, `UserUpdatedEvent`, `UserFullInvalidationEvent`, `UserCacheInvalidationEvent`, `CacheKeyInvalidatedEvent`. Every write path fires the matching event so **peer cluster nodes evict the stale entry** — this is an L1-cache-per-node model with cluster-wide invalidation messaging, not full cache replication.

## 7.5 Clustering (Infinispan module)

- `org.keycloak.connections.infinispan` — connects to/configures the Infinispan `EmbeddedCacheManager`.
- `org.keycloak.cluster.infinispan` — `InfinispanClusterProvider(+Factory)`, implementing the `server-spi-private` `ClusterProvider` (cross-node distributed locks, "execute exactly once across the cluster" semantics via `executeIfNotExecuted`). A `DatabaseAwareClusterProvider(+Factory)` also exists — a DB-coordination alternative to Infinispan-based clustering (relevant for external-Infinispan or no-cluster Quarkus deployments).
- `org.keycloak.models.sessions.infinispan` — **user/client/auth sessions live primarily in Infinispan distributed caches** for cross-node availability: `InfinispanUserSessionProvider(+Factory)`, `UserSessionAdapter`, `AuthenticatedClientSessionAdapter`, `InfinispanAuthenticationSessionProvider(+Factory)`, `InfinispanUserLoginFailureProvider` (brute-force counters, chapter 3), `InfinispanSingleUseObjectProvider`, `InfinispanRevokedTokenProvider`. A separate **`PersistentUserSessionProvider`** implements the newer "persistent user sessions" feature, additionally/alternatively persisting sessions to the relational DB for HA/restart-survival instead of relying purely on in-memory Infinispan replication.
- `org.keycloak.jgroups` — JGroups transport/protocol classes for Infinispan cluster discovery and membership.

## 7.6 Database migration (Liquibase)

`model/jpa/src/main/java/org/keycloak/connections/jpa/updater/liquibase/`:

- `LiquibaseJpaUpdaterProvider(+Factory)` implements `JpaUpdaterProvider`/`JpaUpdaterSpi`, invoking Liquibase's `update()` against the master changelog.
- Master changelog: `model/jpa/src/main/resources/META-INF/jpa-changelog-master.xml`, `<include>`-ing one versioned changelog file per release that changed schema (`jpa-changelog-1.0.0.Final.xml` ... `jpa-changelog-26.3.0.xml`), plus DB-specific variants (e.g. `-db2.xml`) and `jpa-changelog-authz-*.xml` for the authorization-services schema.
- **Cross-node boot locking**: `lock/LiquibaseDBLockProvider(+Factory)`, plus custom Liquibase lock-table SQL generators (`CustomLockDatabaseChangeLogStatement`, `CustomInsertLockRecordGenerator`, with MySQL-specific overrides) — this is the same `DBLockProvider` mechanism referenced in chapter 2's boot sequence, ensuring only one node runs migrations on a multi-node first boot/upgrade.
- Migrations run automatically at boot, before the app serves traffic; they can also be exported to raw SQL for a DBA to review/apply manually instead.

## 7.7 Mental model to carry forward

For *any* domain object in this codebase, you can always answer "where does
this live" by asking three questions in order:

1. **What SPI interface is it?** (`server-spi`/`server-spi-private`, e.g. `RealmModel`, `UserSessionModel`, `Resource`).
2. **What's the default provider chain for it?** — usually
   `KeycloakSession accessor → Infinispan cache decorator → JPA provider → Hibernate → RDBMS`
   for realm/client/user/role/group data, or
   `KeycloakSession accessor → Infinispan distributed cache (± DB persistence)`
   for session/auth-session/brute-force/single-use-object data.
3. **Does a write need to be seen cluster-wide?** — if yes, look for the matching `*Event` in `cache/infinispan/events/` fired on that write path.

Continue to
[08-admin-api-events-operator.md](08-admin-api-events-operator.md).
