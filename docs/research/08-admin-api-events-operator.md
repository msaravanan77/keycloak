# 8. Admin REST API, Account Console, Events, and the Kubernetes Operator

## 8.1 Admin REST API entry point

**`org.keycloak.services.resources.admin.AdminRoot`**
(`services/.../resources/admin/AdminRoot.java`, `@Path("/admin")`) is the
sibling root resource to `RealmsResource` (chapter 2). Flow:

1. `AdminRoot.authenticateRealmAdminRequest(session)` (static) extracts the bearer token from the `Authorization` header (`AppAuthManager.extractAuthorizationHeaderToken`), decodes the JWT to find the **issuer realm**, then runs `AppAuthManager.BearerTokenAuthenticator` against that realm to validate the token and produce an `AuthenticationManager.AuthResult`, wrapped as **`AdminAuth`** (realm, token, user, client).
2. `@Path("realms") getRealmsAdmin()` calls the above, returns `RealmsAdminResource(session, auth, tokenManager)`.
3. **`RealmsAdminResource`** — `@Path("{realm}") getRealmAdmin(name)` looks up the target `RealmModel`, checks the caller's *own* token realm is either the master/administration realm or matches the target realm (else 403), builds `AdminPermissionEvaluator realmAuth = AdminPermissions.evaluator(session, realm, auth)`, and returns `RealmAdminResource(session, realmAuth, adminEvent)`.
4. **`RealmAdminResource`** is the big per-realm dispatcher — everything under `/admin/realms/{realm}/...` hangs off it as sub-resource locators: `clients` → `ClientsResource`, `client-scopes`/`client-templates` → `ClientScopesResource`, `roles` → `RoleContainerResource`, `roles-by-id` → `RoleByIdResource`, `users` → `UsersResource`, `groups` → `GroupsResource`, `identity-provider` → `IdentityProvidersResource`, `authentication` → `AuthenticationManagementResource`, `events`/`admin-events`(+`/config`), `components` → `ComponentResource`, `attack-detection` → `AttackDetectionResource`, `organizations` → `OrganizationsResource`, `partialImport`/`partial-export`, `client-registration-policy`, `clients-initial-access`, `localization`.

Every sub-resource is a thin JAX-RS wrapper that reads/writes exactly the
`server-spi` model objects and provider accessors from chapter 7 — the
Admin REST API has no separate domain model of its own.

## 8.2 Admin authorization model

Two layers, both gate-checked before a sub-resource acts:

1. **Coarse RBAC via realm-management client roles** —
   `org.keycloak.models.AdminRoles` defines `admin`, `create-realm`,
   `realm-admin`, `view-*`/`manage-*` (`manage-users`, `manage-realm`,
   `manage-clients`, `manage-events`, `manage-identity-providers`,
   `manage-authorization`, `manage-organizations`), `query-*`,
   `impersonation` — realm roles on each realm's `realm-management` client
   (or the master realm's `<realm>-realm` client, for cross-realm admins).
2. **Fine-Grained Admin Permissions (FGAP)** —
   `services/.../resources/admin/fgap/`. `AdminPermissions.evaluator(session,
   realm, auth)` returns an `AdminPermissionEvaluator`, implemented by
   `MgmtPermissions` (classic role-based check) unless
   `Profile.Feature.ADMIN_FINE_GRAINED_AUTHZ_V2` is enabled, in which case
   `MgmtPermissionsV2` is used — backed by `FineGrainedAdminPermissionEvaluator`
   and the Authorization Services engine from chapter 6. Per-entity
   evaluators (each with a `*V2` counterpart plus a
   `*PermissionManagement` CRUD-on-permissions interface):
   `UserPermissionEvaluator`, `ClientPermissionEvaluator`,
   `GroupPermissionEvaluator`, `RolePermissionEvaluator`,
   `RealmPermissionEvaluator`, `IdentityProviderPermissions`,
   `OrganizationPermissionEvaluator`. Sub-resources call methods like
   `realmAuth.users().canManage()` before acting.
   `AdminPermissions.registerListener` hooks role/client/group removal to
   clean up any associated FGAP permission policies.

## 8.3 Account Console backend

Entry: **`AccountLoader`** (`.../resources/account/AccountLoader.java`,
`@Path("/")` and `@Path("{version:v\\d...}")`) dispatches to
**`AccountRestService`**, gated by `AccountRoles`
(`MANAGE_ACCOUNT`, `VIEW_PROFILE`, ...) via `auth.requireOneOf(...)`/`auth.require(...)`:

| Path | Purpose |
|---|---|
| `/` (GET/POST) | Profile view/update, via `UserProfileProvider` |
| `/credentials` → `AccountCredentialResource` | Credential management (password, OTP, WebAuthn) |
| `/sessions` → `SessionResource` | List/revoke the user's own sessions |
| `/linked-accounts` → `LinkedAccountsResource` | Manage identity-provider links (chapter 4) |
| `/applications`(+`/{clientId}/consent`, `/sessions`) | Consent grant/revoke/update, per-app session revocation |
| `/organizations`, `/verifiable-credentials`, `/groups`, `/resources`, `/supportedLocales` | Feature-specific sub-resources |

This is the self-service surface a logged-in end user (not an admin) uses.

## 8.4 Events SPI

Interfaces, `server-spi-private/src/main/java/org/keycloak/events/`:
`EventListenerProvider(+Factory)`/`EventListenerSpi` (login/user events),
`EventStoreProvider(+Factory)`/`EventStoreSpi` (persistence), `Event`,
`EventBuilder`, `EventType`; plus `admin/AdminEvent`, `admin/OperationType`,
`admin/ResourceType`, `admin/AuthDetails` for the separate admin-audit event
stream.

**Dispatch**: `EventBuilder.send(...)` checks
`realm.getEnabledEventTypesStream()`, then `sendNow(...)`: writes to
`EventStoreProvider.onEvent(event)` if a store is configured and the event
type is enabled, then loops over every listener from
`realm.getEventsListenersStream()` calling `listener.onEvent(event)` —
each wrapped in try/catch so one misbehaving listener can't break another.

**Default persistence**: `JpaEventStoreProvider(+Factory)`
(`model/jpa/.../events/jpa/`, entities `EventEntity`/`AdminEventEntity`,
query builders `JpaEventQuery`/`JpaAdminEventQuery`) — the relational DB,
*not* Infinispan (the `events` package under `model/infinispan/.../cache/
infinispan/events/` is the unrelated cache-invalidation messaging from
chapter 7, not the login/admin audit log). `model/jpa/.../events/outbox/*`
(`OutboxStore`, `OutboxDrainerTask`, `OutboxDeliveryHandler`) implements a
transactional outbox for reliable event delivery.

**Built-in listeners**: `events/log/JBossLoggingEventListenerProvider(+Factory)`
(server log), `events/email/EmailEventListenerProvider(+Factory)` (emails
the user on certain events), and
`quarkus/runtime/.../metrics/events/MicrometerUserEventMetricsEventListenerProviderFactory`
(exposes event counters as Micrometer metrics).

**Admin events** specifically go through
`org.keycloak.services.resources.admin.AdminEventBuilder`, constructed
per-request in `RealmsAdminResource.getRealmAdmin` and threaded through
`RealmAdminResource` — every mutating admin API call is auditable this way.

## 8.5 The Kubernetes Operator (`operator/`)

Architecturally **separate** from everything above — it never touches the
runtime authentication/token path directly. It's a pure-Java Quarkus
application built on the **Java Operator SDK** (Fabric8 Kubernetes client),
i.e. a standard Kubernetes controller that manages Keycloak server
*deployments*, not Keycloak *logins*.

- **CRDs** (`org.keycloak.operator.crds.v2beta1.deployment`): `Keycloak`, `KeycloakSpec`, `KeycloakStatus`(+`Aggregator`/`Condition`) — the top-level "run a Keycloak cluster like this" resource. `crds.v2beta1.realmimport`: `KeycloakRealmImport`(+`Spec`/`Status`) — declarative realm-JSON import as a CR. `crds.v2alpha1.client`: `KeycloakOIDCClient`/`KeycloakSAMLClient` — declarative client registration CRs.
- **Reconciler**: `org.keycloak.operator.controllers.KeycloakController` (`@ControllerConfiguration` + `@Workflow(...)`, `implements Reconciler<Keycloak>`), orchestrating `DependentResource`s: `KeycloakDeploymentDependentResource` (StatefulSet/Deployment), `KeycloakServiceDependentResource`, `KeycloakDiscoveryServiceDependentResource`, `KeycloakIngressDependentResource`, `KeycloakNetworkPolicyDependentResource`, `KeycloakServiceMonitorDependentResource`, `KeycloakAdminSecretDependentResource`, `KeycloakUpdateJobDependentResource`.
- **Companion controllers**: `KeycloakRealmImportController` (+ `KeycloakRealmImportJobDependentResource`/`SecretDependentResource`) runs a Kubernetes Job to import a realm when a `KeycloakRealmImport` CR is created; `KeycloakOIDCClientController`/`KeycloakSAMLClientController` reconcile client CRs **against the running Keycloak's own Admin REST API** (§8.1–8.2) — this is the one place the operator does call into the runtime server, but as an ordinary admin API client, not as a special code path.
- `KeycloakDistConfigurator` translates the CR spec into Keycloak server config/CLI args (chapter 2's config sources). `org.keycloak.operator.update` implements rolling/recreate update strategies.

**Takeaway**: the operator is the Kubernetes-native *lifecycle management*
layer (install, configure, upgrade, bootstrap realms/clients) sitting
outside and above the server described in chapters 1–7 — think "the thing
that runs `kc.sh` for you declaratively," not "part of the auth server."

## 8.6 Frontend orientation (non-Java, for context only)

The Admin Console UI (`js/apps/admin-ui`) and Account Console UI
(`js/apps/account-ui`) are separate TypeScript/React single-page
applications that consume the REST APIs in §8.1–8.3 over HTTP via
`js/libs/keycloak-admin-client`. They contain no authentication or
authorization logic of their own — every access decision is enforced
server-side by the Java code documented in this knowledge base.

---

This completes the eight-chapter tour. For quick term lookups, see
[glossary.md](glossary.md); for the two architecture diagrams, see
[diagrams/](diagrams/).
