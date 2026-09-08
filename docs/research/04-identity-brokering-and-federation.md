# 4. Identity Brokering & User Federation (Multiple Identity Sources)

Keycloak supports users whose credentials/records live in three fundamentally
different places, and this chapter is the map of all three:

| Source | Who authenticates the user? | Where do user attributes live? | SPI |
|---|---|---|---|
| **Local** | Keycloak itself (password/OTP/WebAuthn against local credentials) | Keycloak's own DB | (chapters 3, 7) |
| **Federated** (LDAP/Kerberos) | Keycloak (against the federated store's credentials) or the KDC (Kerberos) | External store, lazily imported/cached as a local shadow `UserModel` | `org.keycloak.storage.UserStorageProvider` |
| **Brokered** (social/OIDC/SAML/another Keycloak) | The *external* IdP entirely — Keycloak never sees the password | Keycloak, but only what the external IdP asserts, linked via `FederatedIdentityModel` | `org.keycloak.broker.provider.IdentityProvider` |

The one-sentence distinction that matters: **federation** means Keycloak
*is* the identity provider/authentication authority but *sources* user data
externally; **brokering** means Keycloak *delegates the authentication
decision itself* to a separate external IdP and then links the result to a
local user. The code structure mirrors this exactly:
`storage.UserStorageProvider` does data lookup/sync;
`broker.provider.IdentityProvider` does `performLogin()`/`callback()`
redirect dances.

## 4.1 Identity Brokering SPI

Core interfaces, `server-spi-private/src/main/java/org/keycloak/broker/provider/`:

- **`IdentityProvider<C extends IdentityProviderModel>`** (`extends Provider`) — `getConfig()`, `callback()`, `export()`, `reloadKeys()`, `isMapperSupported()`.
- **`IdentityProviderFactory<T>`** (`extends ProviderFactory` + `ConfiguredProvider`) — `create(session, IdentityProviderModel model)`, `createConfig()`, `parseConfig()`.
- **`AbstractIdentityProvider<C>`** — base class with shared defaults (token-exchange helpers, `getLinkingUrl()`, email/verification helpers).

Per-realm config is **`IdentityProviderModel`**
(`server-spi/.../models/IdentityProviderModel.java`): `alias`, `providerId`,
`enabled`, `trustEmail`, `storeToken`, `firstBrokerLoginFlowId`,
`postBrokerLoginFlowId`, `hideOnLogin`, `organizationIds`, and a free-form
`Map<String,String> config` (client id/secret, URLs, ...) — this is exactly
what an admin fills in on the "Identity Providers" screen.

Concrete implementations (`services/src/main/java/org/keycloak/broker/` and `.../social/`):

- **Generic OIDC**: `broker/oidc/OIDCIdentityProvider(+Factory)`, config `OIDCIdentityProviderConfig`, built on `broker/oidc/AbstractOAuth2IdentityProvider`.
- **Keycloak-to-Keycloak**: `broker/oidc/KeycloakOIDCIdentityProvider(+Factory)`.
- **SAML**: `broker/saml/SAMLIdentityProvider(+Factory)`, `broker/saml/SAMLEndpoint` (the ACS/callback endpoint), `SAMLIdentityProviderConfig`.
- **Social** (all `extends AbstractOAuth2IdentityProvider`, under `social/<name>/`): Google, GitHub, GitLab, Facebook, Twitter, Microsoft, LinkedIn, Bitbucket, Instagram, OpenShift, PayPal, StackOverflow.
- Other broker types present: `broker/kubernetes`, `broker/spiffe`, `broker/oid4vp`, `broker/jwtauthorizationgrant`, `broker/trust` (X.509/trust-material), `broker/oauth` (generic OAuth2).

## 4.2 Brokered login — the redirect/callback dance

Entry point: **`org.keycloak.services.resources.IdentityBrokerService`**
(`services/.../resources/IdentityBrokerService.java`, ~1600 lines).

1. `GET/POST /{realm}/broker/{provider_alias}/login` → `performLogin()` resolves the `IdentityProviderModel`, gets the `IdentityProvider` instance from its factory, calls `identityProvider.performLogin(request)`.
2. `AbstractOAuth2IdentityProvider.performLogin()` → `createAuthorizationUrl()` builds the redirect URL to the external IdP's own authorization endpoint (state, PKCE, scope, redirect_uri) and returns HTTP 302. **The end user's browser now leaves Keycloak entirely.**
3. The external IdP authenticates the user by whatever means it wants, then redirects back to `GET /{realm}/broker/{provider_alias}/endpoint` → `IdentityBrokerService.getEndpoint()` → `identityProvider.callback(realm, this, event)`.
4. `AbstractOAuth2IdentityProvider.callback()` hands off to its inner `Endpoint` class's `@GET authResponse(state, code, error, error_description)`: validates `state` against the stored authentication session, exchanges the authorization `code` for tokens directly with the external IdP (server-to-server), builds a **`BrokeredIdentityContext`**, and calls `callback.authenticated(context)`. (For SAML, the analogous inbound Assertion Consumer Service is `SAMLEndpoint`.)
5. **`IdentityBrokerService.authenticated(BrokeredIdentityContext context)`** is where matching happens: it builds a `FederatedIdentityModel(providerAlias, context.getId(), context.getUsername(), context.getToken())` and calls `session.users().getUserByFederatedIdentity(realm, federatedIdentityModel)`.
   - **Already linked** (a local user was previously linked to this external identity) → `validateUser()`, `updateFederatedIdentity()`, mark that user authenticated on the auth session, then `finishOrRedirectToPostBrokerLogin()` — either finish immediately or run the realm's configured **post-broker-login flow** (e.g. to enforce MFA locally even though the external IdP already authenticated the user).
   - **No local match yet** → stash the serialized `BrokeredIdentityContext` on the auth session, reset the flow to `LoginActionsService.FIRST_BROKER_LOGIN_PATH`, and redirect into the realm's **first-broker-login flow**.

## 4.3 First-broker-login: linking a brokered identity to a local account

`AbstractIdpAuthenticator`
(`services/.../authentication/authenticators/broker/AbstractIdpAuthenticator.java`)
is the base class for every authenticator used in the first-broker-login
flow — it reads the stashed `BrokeredIdentityContext` off the auth session
and dispatches to subclasses. Concrete steps (same package):

- `IdpDetectExistingBrokerUserAuthenticator` — is there already a local user with a matching email/username?
- `IdpConfirmLinkAuthenticator` / `IdpConfirmOverrideLinkAuthenticator` — ask the user to confirm linking to that existing account.
- `IdpCreateUserIfUniqueAuthenticator` — auto-create a brand-new local user if there's no collision.
- `IdpEmailVerificationAuthenticator`, `IdpReviewProfileAuthenticator`, `IdpUsernamePasswordForm`, `IdpAutoLinkAuthenticator` — verification / profile-completion / auto-link policy variants, all realm-configurable like any other flow.

Once first-broker-login completes, `IdentityBrokerService.afterFirstBrokerLogin()`
resumes and **persists** the `FederatedIdentityModel` row (the actual
link), then proceeds to `finishOrRedirectToPostBrokerLogin()` exactly as in
the "already linked" path above. A logged-in user explicitly choosing
"link account" from the Account Console (rather than logging in) goes
through the same service's `performAccountLinking()`, detected via
`isDoingAccountLinking()`.

## 4.4 User Federation SPI (LDAP/Kerberos)

The base contracts live in **`model/storage`** (not `server-spi`/`server-spi-private`
as you might expect for something this central):

- `UserStorageProvider` (`model/storage/src/main/java/org/keycloak/storage/`) — a marker interface with an `EditMode` (`READ_ONLY` / `WRITABLE` / `UNSYNCED`), extended à la carte by capability interfaces: `UserLookupProvider`, `UserQueryProvider`/`UserQueryMethodsProvider`, `UserRegistrationProvider`, `UserCountMethodsProvider`, `UserBulkUpdateProvider` (server-spi), plus `DatastoreProvider`/`OnCreateComponent` (server-spi-private).
- **Dispatch**: `model/storage-private/src/main/java/org/keycloak/storage/UserStorageManager.java` implements `getUserById/getUserByUsername/getUserByEmail` by parsing the composite user ID via `StorageId` — plain (unprefixed) IDs go straight to the local JPA store; provider-prefixed IDs route to the matching federation provider instance (resolved per-realm component) implementing `UserLookupProvider`.
- **Caching**: `UserCache`/`CachedUserModel` (`model/storage/.../models/cache/`) define the cache contract; the Infinispan implementation **`UserCacheSession`** (`model/infinispan/.../models/cache/infinispan/UserCacheSession.java`) sits in front of `UserStorageManager` — checks the Infinispan cache by realm/username/email first, only calling out to LDAP (or the JPA store) on a miss, then caches the result as a `CachedUserModel`. This is *why* LDAP-backed logins don't hit LDAP on every single request — see chapter 7 for the cache-invalidation mechanics.

### LDAP (`federation/ldap/src/main/java/org/keycloak/storage/ldap/`)

- **`LDAPStorageProvider`** — the actual `UserStorageProvider` implementation. `getUserByUsername()` queries LDAP then calls `importUserFromLDAP()` (creates/updates the local shadow `UserModel` row from the LDAP attributes). `proxy()` wraps the imported user in a delegating `UserModel` that, for `WRITABLE` mode, writes attribute changes straight back to LDAP, running each configured `LDAPStorageMapper.proxy()` in turn.
- **`LDAPIdentityStoreRegistry`** — caches/creates `LDAPIdentityStore` connections per `ComponentModel` (i.e. per configured LDAP provider instance) — connection pooling.
- **`LDAPUtils`**, **`LDAPOperationManager`**, **`LDAPContextManager`**, **`LDAPQuery`/`LDAPQueryConditionsBuilder`** — low-level JNDI query/connection plumbing.
- **Mapper SPI**: `LDAPStorageMapper` (`mappers/LDAPStorageMapper.java`) — `syncDataFromFederationProviderToKeycloak()`, `syncDataFromKeycloakToFederationProvider()`, plus a per-attribute `proxy()` contribution. Concrete mappers: `GroupLDAPStorageMapper`, `HardcodedLDAPRoleStorageMapper`, `FullNameLDAPStorageMapper`, `CertificateLDAPStorageMapper`, `KerberosPrincipalAttributeMapper`, orchestrated by `LDAPStorageMapperManager` in an order decided by `LDAPMappersComparator`.

### Kerberos / SPNEGO (`federation/kerberos/src/main/java/org/keycloak/federation/kerberos/`)

- `KerberosFederationProvider` (`implements UserStorageProvider`) — authenticates against a KDC and, in LDAP-backed deployments, defers actual user-data lookup to the LDAP provider.
- `KerberosServerSubjectAuthenticator` / `KerberosUsernamePasswordAuthenticator` — JAAS `Subject` login against the KDC.
- `SPNEGOAuthenticator` — validates a SPNEGO/GSS token carried in a `Negotiate` `Authorization` header.
- Browser-flow integration point:
  `services/.../authentication/authenticators/browser/SpnegoAuthenticator(+Factory)`
  — the actual `Authenticator` wired into the `browser` flow that issues
  the `WWW-Authenticate: Negotiate` challenge and, on success, resolves the
  Kerberos principal into a local `UserModel` via the federation provider
  above. This is a good example of how "an entirely different auth
  mechanism" is still just one more `Authenticator` plugged into the same
  engine from chapter 3.

## 4.5 Putting it together: three identity sources, one engine

However a user is authenticated — local password form, LDAP-backed password
form, Kerberos/SPNEGO negotiation, or a full external-IdP redirect via
brokering — the *outcome* the rest of the system cares about is identical:
an `AuthenticationSessionModel` ends up with `getAuthenticatedUser()` set to
some `UserModel`, and `AuthenticationProcessor.authenticationComplete()`
takes over from there exactly as described in chapter 3, ultimately handing
off to a `LoginProtocol` for token/assertion issuance (chapter 5). Brokering
is the one path that additionally passes through `IdentityBrokerService`
*before* re-entering the normal flow engine (for first/post-broker-login),
but it rejoins the same trunk. See
[diagrams/auth-sequence-multi-idp.svg](diagrams/auth-sequence-multi-idp.svg)
for this convergence drawn out explicitly.

Continue to [05-oidc-and-saml-protocols.md](05-oidc-and-saml-protocols.md).
