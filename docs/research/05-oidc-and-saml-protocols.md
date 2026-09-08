# 5. OIDC & SAML Protocols: Endpoints and Token/Assertion Issuance

Chapter 3 ended at `AuthenticationManager.redirectAfterSuccessfulFlow(...)`.
This chapter covers what's on the other side of that call, plus the wire
endpoints clients actually talk to.

## 5.1 `LoginProtocol` — the abstraction shared by OIDC and SAML

`server-spi-private/src/main/java/org/keycloak/protocol/LoginProtocol.java`
(`interface LoginProtocol extends Provider`):

```java
setSession(...); setRealm(...); setUriInfo(...); setHttpHeaders(...); setEventBuilder(...);  // fluent config
authenticated(AuthenticationSessionModel, UserSessionModel, ClientSessionContext);           // flow-complete hook
sendError(AuthenticationSessionModel, Error, String);
sendError(ClientModel, ClientData, Error);   // overload for when the auth session is already gone
backchannelLogout(...); frontchannelLogout(...); finishBrowserLogout(...);
requireReauthentication(...);
getClientData(...);   // protocol-specific data needed for error responses (redirect_uri+state for OIDC, RelayState for SAML)
```

`LoginProtocolFactory` (same package) provides
`createProtocolEndpoint(session, event)` (the JAX-RS root resource for that
protocol) and `getBuiltinMappers()`.

Implementations: `OIDCLoginProtocol(+Factory)`
(`services/.../protocol/oidc/`) and `SamlProtocol(+Factory)`
(`services/.../protocol/saml/`).

The glue: `AuthenticationManager.redirectAfterSuccessfulFlow(...)`
(`services/.../services/managers/AuthenticationManager.java`) looks up
`session.getProvider(LoginProtocol.class, authSession.getProtocol())` and
calls `protocol.authenticated(authSession, userSession, clientSessionCtx)`
— returning the protocol-specific HTTP response (a redirect carrying an
authorization `code` for OIDC, a SAML `Response` via POST/Redirect binding
for SAML). **This one call is the fork in the road between §5.2 and §5.6
below.**

## 5.2 OIDC endpoints

`services/.../protocol/oidc/OIDCLoginProtocolService.java` is the JAX-RS
sub-resource mounted at `/realms/{realm}/protocol/openid-connect`:

| Path | Class | Purpose |
|---|---|---|
| `/auth`, `/auth/device` | `AuthorizationEndpoint` (`endpoints/AuthorizationEndpoint.java`) | Authorization endpoint — starts the flow engine (chapter 3). |
| `/token` | `TokenEndpoint` (`endpoints/TokenEndpoint.java`) | Token endpoint — every grant type. |
| `/token/introspect` | `TokenIntrospectionEndpoint` | RFC 7662 introspection. |
| `/certs` | — | JWKS (`JSONWebKeySet` via `JWKSServerUtils.getRealmJwks`). |
| `/userinfo` | `UserInfoEndpoint` | UserInfo (GET+POST). |
| `/logout` | `LogoutEndpoint` | RP-initiated logout. |
| `/revoke` | `TokenRevocationEndpoint` | RFC 7009 revocation. |
| `/login-status-iframe.html`, `/3p-cookies` | — | Session-status iframe helpers for browser SSO. |
| `/ext/{extension}` | pluggable `OIDCExtProvider` | Extension point (e.g. PAR). |

Device/CIBA/PAR live as their own mini-services:
`protocol/oidc/grants/device/endpoints/DeviceEndpoint.java`,
`protocol/oidc/grants/ciba/endpoints/{BackchannelAuthenticationEndpoint,
CibaRootEndpoint,BackchannelAuthenticationCallbackEndpoint}.java`,
`protocol/oidc/par/...`. The `/.well-known/openid-configuration` discovery
document is a separate SPI, `OIDCWellKnownProvider(+Factory)`.

## 5.3 The token endpoint and grant types

`TokenEndpoint.processGrantRequest()` parses form params, reads
`grant_type`, and resolves the concrete implementation through a pluggable
SPI: `session.getProvider(OAuth2GrantType.class, grantType)`. Implementations
under `protocol/oidc/grants/`:

| Grant | Class |
|---|---|
| Authorization Code | `AuthorizationCodeGrantType` |
| Refresh Token | `RefreshTokenGrantType` |
| Client Credentials | `ClientCredentialsGrantType` |
| Resource Owner Password Credentials ("direct access grant") | `ResourceOwnerPasswordCredentialsGrantType` |
| Device Code | `DeviceGrantType` |
| CIBA | `CibaGrantType` |
| Token Exchange | `TokenExchangeGrantType` |
| JWT Bearer Authorization Grant | `JWTAuthorizationGrantType` |
| UMA2 ticket | `PermissionGrantType` (chapter 6) |
| Pre-authorized code (OID4VC) | `PreAuthorizedCodeGrantType` |

Each grant runs `grant.preProcess()` (client-policy hooks) then
`grant.process(context)`, where `context` bundles session/client
config/form params/event/CORS/`tokenManager`. Common logic lives in
the abstract base **`OAuth2GrantTypeBase`**:
`createTokenResponseBuilder(...)` obtains
`TokenManager.AccessTokenResponseBuilder responseBuilder =
tokenManager.responseBuilder(realm, client, event, session, userSession,
clientSessionCtx).accessToken(token)`, then conditionally
`.generateRefreshToken()` and `.generateIDToken().generateAccessTokenHash()`;
`createTokenResponse(...)` calls `responseBuilder.build()`.

### `TokenManager` — the actual token minter

`services/.../protocol/oidc/TokenManager.java` (~1750 lines, a concrete
class, not an SPI) mints tokens:

- `createClientAccessToken(...)` — builds the raw `AccessToken`.
- `initToken(...)` — standard claims (`iss`, `sub`, `exp`, `sid`, `azp`, `aud`, ...).
- `transformAccessToken` / `transformIDToken` / `generateUserInfoClaims` / `transformIntrospectionAccessToken` — run the client's `ProtocolMapperModel`s (`org.keycloak.protocol.ProtocolMapper` SPI, impls under `protocol/oidc/mappers/*` — e.g. `UserAttributeMapper`, `UserRealmRoleMappingMapper`, `AudienceProtocolMapper`) to add/shape claims per requested scope.
- Inner class `AccessTokenResponseBuilder` — `generateAccessToken()`, `generateRefreshToken()`, `generateIDToken()`; `build()` calls `session.tokens().encode(accessToken)` for the JWT string, `session.tokens().encodeAndEncrypt(idToken)`, `session.tokens().encode(refreshToken)`, then `transformAccessTokenResponse(...)` before returning the `AccessTokenResponse`.

### Signing (`session.tokens()`)

The actual encode/sign step is a *different*, smaller SPI:
`server-spi/.../models/TokenManager.java`, obtained via `session.tokens()`,
implemented by **`DefaultTokenManager`**
(`services/.../jose/jws/DefaultTokenManager.java`). `encode(Token token)`:
picks the signature algorithm for the token's category, gets
`SignatureProvider signatureProvider = session.getProvider(SignatureProvider.class, alg)`,
calls `signatureProvider.signer()` → `SignatureSignerContext`, then
`new JWSBuilder().type(type).jsonContent(token).sign(signer)`. Decoding
mirrors this with `signatureProvider.verifier(kid)`. Realm signing keys
themselves come from the `KeyManager`/`KeyProvider` SPI (`session.keys()`),
backed by realm `ComponentModel`s (rsa-generated, hmac-generated,
java-keystore, ...).

## 5.4 Client authentication (at the token endpoint)

SPI: `server-spi-private/.../authentication/ClientAuthenticator.java` —
`authenticateClient(ClientAuthenticationFlowContext context)`.
Implementations (`services/.../authentication/authenticators/client/`):
`ClientIdAndSecretAuthenticator` (`client_secret_post`/`client_secret_basic`),
`JWTClientAuthenticator` (`private_key_jwt`), `JWTClientSecretAuthenticator`
(`client_secret_jwt`), `X509ClientAuthenticator` (`tls_client_auth`/mTLS),
`AttestationBasedClientAuthenticator`. `TokenEndpoint.checkClient()` calls
`AuthorizeClientUtil.authorizeClient(...)`, which iterates the client's
configured `ClientAuthenticator`s. This reuses the *same*
`ClientAuthenticationFlow` concept from chapter 3 — client auth is modeled
as its own flow (`CLIENT_AUTHENTICATION_FLOW = "clients"`).

## 5.5 UserInfo & introspection

`UserInfoEndpoint` — validates the bearer access token, returns
`TokenManager.generateUserInfoClaims(...)` (plain JSON or a signed JWT, per
client config). `TokenIntrospectionEndpoint` — delegates to a pluggable
`TokenIntrospectionProvider` SPI; built-ins are
`AccessTokenIntrospectionProvider(+Factory)` and
`RefreshTokenIntrospectionProvider(+Factory)`.

## 5.6 SAML endpoints and assertion building

`services/.../protocol/saml/SamlService.java`
(`extends AuthorizationEndpointBase`) is mounted at
`/realms/{realm}/protocol/saml`. Notable operations:

- Root path (`@GET`/`@POST`) dispatches an inner binding-handling routine supporting both POST and Redirect bindings — handles both SP-initiated authentication requests and IdP responses to logout.
- `descriptor()` — `GET /descriptor`, publishes IdP metadata (`IDPMetadataDescriptor`).
- `idpInitiatedSSO(clientUrlName, relayState)` — `GET /clients/{client}`, IdP-initiated SSO.
- `artifactResolutionService(...)` — SOAP artifact binding.
- `soapBinding(...)` — ECP/SOAP binding.

**Assertion building/signing**: `SamlProtocol.authenticated(...)`
(the SAML `LoginProtocol` implementation) constructs a
**`org.keycloak.saml.SAML2LoginResponseBuilder`**
(`saml-core/src/main/java/org/keycloak/saml/SAML2LoginResponseBuilder.java`)
— sets request ID, destination, issuer, assertion/session expirations,
session index — then runs `SAMLAttributeStatementMapper` /
`SAMLRoleListMapper` / `SAMLNameIdMapper` protocol mappers
(`protocol/saml/mappers/*`) before `buildDocument(samlModel)`. Signing and
encryption go through **`JaxrsSAML2BindingBuilder`**
(`protocol/saml/JaxrsSAML2BindingBuilder.java`):
`.signatureAlgorithm(...).signWith(keyName, privateKey, publicKey,
certificate).signDocument()` and/or `.signAssertions()`, per
`SamlClient.requiresRealmSignature()/requiresAssertionSignature()/requiresEncryption()`
— the low-level XML-DSig work (`SAML2Signature`, `XMLSignatureUtil`) lives
in the `saml-core` module. Logout uses the same builder machinery.

## 5.7 Sessions ↔ tokens

`UserSessionModel` (chapter 7) is the browser SSO session; each client the
user has touched during that session gets its own
`AuthenticatedClientSessionModel` (created/attached via
`TokenManager.attachAuthenticationSession(...)`). Access/refresh/ID tokens
embed `sid` (session id) and are validated against the live client session
on refresh (`RefreshTokenGrantType` re-checks `AuthenticatedClientSessionModel`).
**Offline tokens** are a distinct persisted variant
(`UserSessionModel.isOffline()`) — the offline grant persists the session so
its refresh token survives normal SSO-session expiry.

## 5.8 End-to-end: the standard authorization_code flow

```
1. Browser  GET /realms/{realm}/protocol/openid-connect/auth
              → OIDCLoginProtocolService.auth() → AuthorizationEndpoint.buildGet()
              → validates client/redirect_uri/scope, creates AuthenticationSessionModel,
                runs the shared flow engine (chapter 3: login form, MFA, consent...)

2. On success → AuthenticationManager.redirectAfterSuccessfulFlow(...)
              → creates/updates UserSessionModel + AuthenticatedClientSessionModel
              → protocol.authenticated(...) → OIDCLoginProtocol builds the redirect
                with `code` (and `state`), the code itself managed by
                OAuth2CodeParser/OAuth2Code (stored server-side, keyed by the code value)

3. Browser  302 → client's redirect_uri?code=...&state=...

4. Client   POST /realms/{realm}/protocol/openid-connect/token
              grant_type=authorization_code&code=...&redirect_uri=...
              → TokenEndpoint.processGrantRequest()
              → AuthorizeClientUtil authenticates the client (§5.4)
              → AuthorizationCodeGrantType validates code/PKCE/redirect_uri,
                loads UserSessionModel/AuthenticatedClientSessionModel,
                builds ClientSessionContext
              → OAuth2GrantTypeBase.createTokenResponseBuilder(...)
              → TokenManager.responseBuilder(...).accessToken(...)
                    .generateRefreshToken().generateIDToken()...build()

5. build()  runs protocol mappers for claims, signs each token via
              session.tokens().encode(...) → DefaultTokenManager → SignatureProvider
              (realm's active signing key), returns AccessTokenResponse JSON
              { access_token, id_token, refresh_token, expires_in, scope, ... }
```

Continue to [06-authorization-services.md](06-authorization-services.md) —
what happens *after* a client has a token and wants fine-grained,
resource-level permission decisions instead of (or in addition to) plain
role checks.
