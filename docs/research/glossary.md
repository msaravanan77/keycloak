# Glossary

Quick reference for terms used throughout this knowledge base, in
Keycloak's specific sense (not always the generic industry definition).

| Term | Meaning in this codebase |
|---|---|
| **Realm** | An isolated tenant/security domain — its own users, clients, roles, groups, IdPs, flows. `RealmModel`. |
| **Client** (Keycloak sense) | A registered application that can request tokens/assertions from a realm — not to be confused with an HTTP client. `ClientModel`. |
| **`KeycloakSession`** | A per-request unit-of-work object (transaction + provider registry) — **not** an HTTP/browser session. |
| **Provider / ProviderFactory / SPI** | Keycloak's own extension-point pattern: an interface (`Provider`), a factory that builds instances (`ProviderFactory`), discovered via `org.keycloak.provider.Spi` + `ProviderManager`. The mechanism behind nearly every pluggable subsystem in this repo. |
| **Authentication flow** | A realm-configured, admin-editable tree of `AuthenticationExecutionModel`s (steps/sub-flows) executed by `AuthenticationProcessor`. Built-ins: `browser`, `direct grant`, `reset credentials`, `registration`, `first broker login`, `clients` (client auth). |
| **Authenticator** | One executable step in a flow (e.g. password form, OTP, WebAuthn, SPNEGO, "redirect to external IdP"). `Authenticator`/`AuthenticatorFactory`. |
| **Required Action** | An obligation a user must complete after successful authentication but before the session finalizes (update password, verify email, register OTP, ...). `RequiredActionProvider`/`Factory`. |
| **Authentication Session** | Multi-request state for an in-progress login (which step you're on, notes, the eventually-authenticated user). `AuthenticationSessionModel`, rooted in `RootAuthenticationSessionModel`, tracked via the `AUTH_SESSION_ID` cookie. |
| **User Session** | The result of a *completed* login — the browser's SSO session. `UserSessionModel`. |
| **Authenticated Client Session** | One per (user session × client) — tracks that client's refresh tokens etc. `AuthenticatedClientSessionModel`. |
| **User Federation** | Sourcing user records from an external store (LDAP, Kerberos, custom) while Keycloak remains the authentication authority. `UserStorageProvider` SPI. |
| **Identity Brokering** | Delegating the *authentication decision itself* to a separate external IdP (social/OIDC/SAML/another Keycloak), then linking the result to a local user. `IdentityProvider` SPI (`org.keycloak.broker.provider`). |
| **`FederatedIdentityModel`** | The persisted link row between a local `UserModel` and a brokered external identity (provider alias + external user id). |
| **First-Broker-Login flow** | Runs the first time a brokered identity has no existing local-user link — decides whether to auto-create, prompt-to-link, or reject. |
| **Post-Broker-Login flow** | Runs after every brokered login (linked or not) — e.g. to enforce local MFA even though the external IdP already authenticated the user. |
| **LoginProtocol** | The abstraction (`LoginProtocol` SPI) that lets the *same* authentication flow engine drive either an OIDC or a SAML response at the end. |
| **Authorization Code / Token Endpoint / Grant Type** | Standard OAuth2/OIDC terms; in code, grant types are pluggable `OAuth2GrantType` providers dispatched by `TokenEndpoint`. |
| **`TokenManager`** | The concrete class (`services/.../protocol/oidc/TokenManager`) that assembles and mints access/ID/refresh tokens, running `ProtocolMapper`s to shape claims. |
| **Protocol Mapper** | A pluggable rule that adds/shapes a claim (OIDC) or attribute/role statement (SAML) on an issued token/assertion. `ProtocolMapper` SPI. |
| **UMA2 / RPT** | User-Managed Access 2.0 — Keycloak's fine-grained authorization protocol. An **RPT** (Requesting Party Token) is an access token with an embedded list of granted resource/scope permissions. |
| **Resource / Scope / Policy / Permission** (Authorization Services) | `Resource` = a protected asset; `Scope` = an action on it; `Policy` = a composable access-control rule (role/user/group/time/js/aggregated/...); a *Permission* object ties resources/scopes to the policies that must pass. |
| **Fine-Grained Admin Permissions (FGAP)** | Reuses the Authorization Services engine (`RolePolicyProvider` + friends) to authorize the **Admin REST API itself**, as an alternative to plain realm-management role checks. |
| **Realm-management roles** | Coarse admin RBAC roles (`admin`, `manage-users`, `manage-clients`, ...) defined on each realm's `realm-management` client. `org.keycloak.models.AdminRoles`. |
| **`ProviderManager`** | Keycloak's own SPI scanner (reads `META-INF/services/org.keycloak.provider.Spi` and factory service files) — the mechanism `KeycloakProcessor` uses at build time to discover which provider classes exist. |
| **Infinispan** | The embedded/clustered in-memory data grid used for (a) an L1 cache-with-invalidation in front of the JPA model provider, and (b) primary storage for user/auth/login-failure sessions across a cluster. |
| **Liquibase** | The schema-migration tool that applies `jpa-changelog-*.xml` files to the relational database at boot, coordinated cluster-wide via `DBLockProvider`. |
| **Operator** | The separate Kubernetes controller (`operator/` module) that declaratively deploys/upgrades Keycloak and imports realms/clients on K8s — talks to the Admin REST API as an ordinary client, not part of the runtime auth path. |
| **Quarkus** | The Java framework Keycloak's server runtime is built on (`quarkus/` modules) — supplies the Vert.x-based HTTP layer, CDI/Arc container, build-time extension model, and native-image support. |
