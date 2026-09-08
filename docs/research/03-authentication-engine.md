# 3. The Authentication Flow Engine

This is the heart of Keycloak: a small, generic, tree-walking execution
engine that drives *every* kind of login (username/password, OTP, WebAuthn,
Kerberos/SPNEGO, "redirect to an external IdP," direct-grant, registration,
credential reset) through the same machinery, realm-configurable without
code changes.

## 3.1 Entry point

`GET/POST /realms/{realm}/protocol/openid-connect/auth` is handled by
**`org.keycloak.protocol.oidc.endpoints.AuthorizationEndpoint`**
(`services/.../protocol/oidc/endpoints/AuthorizationEndpoint.java`,
`buildGet()`/`buildPost()`), which extends
**`org.keycloak.protocol.AuthorizationEndpointBase`**
(`services/.../protocol/AuthorizationEndpointBase.java`). The base class owns
`createProcessor(authSession, flowId, flowPath)` and
`handleBrowserAuthenticationRequest(...)`:

1. Resolve the realm's `browser` `AuthenticationFlowModel`.
2. Build an `AuthenticationProcessor`, `setFlowPath(LoginActionsService.AUTHENTICATE_PATH)`.
3. Call `processor.authenticate()`.

Every subsequent form POST during a multi-step login (password page, OTP
page, "confirm link this account," ...) is served by
**`org.keycloak.services.resources.LoginActionsService`**
(paths: `AUTHENTICATE_PATH = "login-actions/authenticate"`,
`REGISTRATION_PATH`, `RESET_CREDENTIALS_PATH`, `FIRST_BROKER_LOGIN_PATH`,
`POST_BROKER_LOGIN_PATH`, `REQUIRED_ACTION`), which **rebuilds a processor
from the persisted `AuthenticationSessionModel`** and calls
`processor.authenticationAction(execution)`. SAML has an analogous entry
point (`SamlService`) that funnels into the same engine.

## 3.2 `AuthenticationProcessor` — the engine

`services/src/main/java/org/keycloak/authentication/AuthenticationProcessor.java`

Key methods:

| Method | Role |
|---|---|
| `authenticateOnly()` | Validates the client session/action, builds a root `AuthenticationFlow` via `createFlowExecution(flowId, null)`, calls `authenticationFlow.processFlow()`. |
| `authenticate()` | `authenticateOnly()`, then if no challenge was produced, `authenticationComplete()`. |
| `authenticationAction(execution)` | Resumes on a form POST for a specific `AuthenticationExecutionModel` — `authenticationFlow.processAction(execution)`. |
| `authenticationComplete()` | After flow success: sets ACR, computes `nextRequiredAction()`. If one exists → `AuthenticationManager.redirectToRequiredActions(...)`. Else → `LoginProtocol.authenticationComplete(authSession)` then `AuthenticationManager.finishedRequiredActions(...)`. |
| `finishAuthentication(LoginProtocol)` | `attachSession()` creates/links the `UserSessionModel` + `ClientSessionContext`, then `AuthenticationManager.redirectAfterSuccessfulFlow(...)`. |
| `createFlowExecution(flowId, execution)` | Dispatches on `AuthenticationFlowModel.getProviderId()` → `DefaultAuthenticationFlow` (basic), `FormAuthenticationFlow`, or `ClientAuthenticationFlow`. |

The inner class `AuthenticationProcessor.Result` implements
`AuthenticationFlowContext` — this is what's passed to every `Authenticator`:
it exposes `success()`, `failure()`, `challenge()` (render a form / redirect
instead of finishing), `attemptedUser`, notes, etc. Every authenticator you
will ever read is essentially "look at `context`, call one of these."

## 3.3 Flows and executions (the configurable tree)

`AuthenticationFlowModel` and `AuthenticationExecutionModel`
(`server-spi/src/main/java/org/keycloak/models/`) are stored **per realm**
(`RealmModel.getAuthenticationFlowById(...)`,
`getAuthenticationExecutionsStream(flowId)`) — this is exactly what the
Admin Console's "Authentication" tab edits.

`AuthenticationExecutionModel` fields:

- `authenticator` — which `Authenticator` factory ID to run.
- `authenticatorConfig` — realm-specific config for that authenticator instance.
- `flowId` / `parentFlow` and `authenticatorFlow` (boolean: is this execution itself a sub-flow?).
- `priority` — execution order within the parent flow.
- `Requirement` — `REQUIRED`, `ALTERNATIVE`, `CONDITIONAL`, `DISABLED`.

Traversal is in **`DefaultAuthenticationFlow.processFlow()` /
`processSingleFlowExecutionModel()`**
(`services/src/main/java/org/keycloak/authentication/DefaultAuthenticationFlow.java`):
walk executions in priority order, recurse into sub-flows, honor
`Requirement` semantics (a `REQUIRED` execution must succeed; an
`ALTERNATIVE` execution succeeding satisfies the group; `CONDITIONAL`
sub-flows only run if their condition-authenticator passes).

Built-in flows are seeded by
**`org.keycloak.models.utils.DefaultAuthenticationFlows`**
(`server-spi-private/.../models/utils/DefaultAuthenticationFlows.java`):

| Constant | Purpose |
|---|---|
| `BROWSER_FLOW = "browser"` | Standard interactive login (cookie SSO check → Kerberos → username/password → OTP/WebAuthn as configured). |
| `DIRECT_GRANT_FLOW = "direct grant"` | Resource Owner Password Credentials grant (no browser/redirect involved). |
| `RESET_CREDENTIALS_FLOW = "reset credentials"` | Forgot-password flow. |
| `REGISTRATION_FLOW` / `REGISTRATION_FORM_FLOW` | Self-registration. |
| `FIRST_BROKER_LOGIN_FLOW` | Runs the first time a brokered identity has no local user match yet (see chapter 4). |
| `CLIENT_AUTHENTICATION_FLOW = "clients"` | How *clients* (not users) authenticate at the token endpoint. |
| `SAML_ECP_FLOW` | SAML Enhanced Client/Proxy profile. |

## 3.4 The `Authenticator` SPI

`server-spi-private/src/main/java/org/keycloak/authentication/Authenticator.java`:

```java
authenticate(AuthenticationFlowContext context);   // initial call — may context.challenge(...)
action(AuthenticationFlowContext context);          // handles the form-submit / callback
requiresUser();
configuredFor(session, realm, user);
setRequiredActions(...);
getRequiredActions(session);
```

`AuthenticatorFactory` extends `ProviderFactory<Authenticator>` +
`ConfigurableAuthenticatorFactory`, discovered like any other SPI factory
(`META-INF/services/org.keycloak.authentication.AuthenticatorFactory`).

Representative built-ins (`services/src/main/java/org/keycloak/authentication/authenticators/`):

- `browser/UsernamePasswordForm.java` (`extends AbstractUsernameFormAuthenticator`) — the password page.
- `browser/OTPFormAuthenticator.java`, `browser/ConditionalOtpFormAuthenticator.java` — TOTP/HOTP.
- `browser/WebAuthnAuthenticator.java` — WebAuthn/FIDO2.
- `directgrant/ValidatePassword.java`, `directgrant/ValidateOTP.java` — non-interactive equivalents used by the direct-grant flow.
- `x509/X509ClientCertificateAuthenticator.java` — mutual-TLS login.
- `conditional/*` — gating conditions for `CONDITIONAL` sub-flows (e.g. "only require OTP if user has an OTP credential configured").
- `broker/*` (e.g. `IdpUsernamePasswordForm`) — steps specific to the first-broker-login flow, see chapter 4.

`AbstractUsernameFormAuthenticator.validateUserAndPassword()` →
`validatePassword()` → checks brute-force status, then
`validateUser()`/credential validation (§3.6).

## 3.5 `RequiredActionProvider` — post-authentication obligations

`server-spi-private/src/main/java/org/keycloak/authentication/RequiredActionProvider.java`:
`evaluateTriggers(context)`, `requiredActionChallenge(context)`,
`processAction(context)`, paired with `RequiredActionFactory`. These are
things a user must do *after* successfully authenticating and *before* a
session is finalized: `UpdatePassword`, `UpdateTotp`, `VerifyEmail`,
`VerifyUserProfile`, `UpdateProfile`, `UpdateEmail`, `TermsAndConditions`,
`WebAuthnRegister`/`WebAuthnPasswordlessRegister`, `RecoveryAuthnCodesAction`,
`DeleteAccount`, `DeleteCredentialAction` (all in
`services/.../authentication/requiredactions/`). Triggered from
`AuthenticationProcessor.nextRequiredAction()` /
`AuthenticationManager.evaluateRequiredActionTriggers()`, served by
`LoginActionsService.requiredActionGET/POST`.

## 3.6 Authentication sessions — state across many requests

A browser login is inherently multi-request (show form → submit → show OTP
form → submit → ...). State is tracked by:

- **`RootAuthenticationSessionModel`**
  (`server-spi/src/main/java/org/keycloak/sessions/RootAuthenticationSessionModel.java`)
  — the entity actually tracked via the `AUTH_SESSION_ID` cookie/query
  param, looked up via `AuthenticationSessionManager`. One root session can
  hold multiple per-tab/per-client sub-sessions (`getAuthenticationSessions()`,
  keyed by tab id).
- **`AuthenticationSessionModel`**
  (`server-spi/src/main/java/org/keycloak/sessions/AuthenticationSessionModel.java`)
  — per-client, per-tab flow state: `getExecutionStatus()`
  (`Map<executionId, ExecutionStatus>`), `getAuthenticatedUser()`,
  `getAuthNote()`/`setAuthNote()` (transient step data such as
  `CURRENT_AUTHENTICATION_EXECUTION`, `CURRENT_FLOW_PATH`), `getClientNote()`,
  and the pending-required-actions set.
- `ClientSessionCode` validates the `code`/action parameter on each request
  in `LoginActionsService`/`AuthenticationProcessor.checkClientSession()`,
  preventing replay/CSRF against the multi-step flow.

## 3.7 Credential validation and brute-force protection

`CredentialProvider<T>` (`server-spi/.../credential/CredentialProvider.java`)
defines credential-type-specific validation (`isValid`), with concrete
implementations for password/OTP/WebAuthn under `services/.../credential/`.
`UserModel.credentialManager()` (`UserCredentialManager`,
`server-spi/.../models/UserCredentialManager.java`) exposes `isValid(...)`
and dispatches to the matching `CredentialProvider`.
`AbstractUsernameFormAuthenticator.validatePassword()` calls
`user.credentialManager().isValid(UserCredentialModel.password(password))`.

Brute-force protection hooks in via
`isDisabledByBruteForce(context, user)` (checked both before and after
password validation), backed by the `BruteForceProtector` SPI
(`server-spi-private/.../services/managers/BruteForceProtector.java`),
implemented by `DefaultBruteForceProtector` /
`DefaultBlockingBruteForceProtector`
(`services/.../services/managers/`), which records failures from login
events and can lock an account out for a configurable window.

## 3.8 Handoff to token issuance

Once the flow (including required actions) is fully satisfied:

```
AuthenticationProcessor.authenticationComplete()
  → session.getProvider(LoginProtocol.class, authSession.getProtocol())
        .authenticationComplete(authSession)                    // protocol-level "done" hook
  → AuthenticationManager.finishedRequiredActions(...)
  → AuthenticationProcessor.finishAuthentication(LoginProtocol)
  → attachSession()   // creates/attaches UserSessionModel + ClientSessionContext
  → AuthenticationManager.redirectAfterSuccessfulFlow(session, realm, userSession,
        clientSessionCtx, request, uriInfo, connection, event, authSession, protocol)
  → protocol.authenticated(authSession, userSession, clientSessionCtx)   // OIDC or SAML — chapter 5
```

`AuthenticationManager.redirectAfterSuccessfulFlow(...)`
(`services/src/main/java/org/keycloak/services/managers/AuthenticationManager.java`)
is the exact seam where the generic flow engine ends and the
protocol-specific response (an OIDC authorization code redirect, or a SAML
Response POST/Redirect) begins — see chapter 5 for what happens on the other
side of that seam. Actual access/ID/refresh token *minting* happens later,
when the client calls the token endpoint with the authorization code.

## 3.9 Sequence recap (browser login, happy path)

1. `AuthorizationEndpoint.buildGet/buildPost` → `handleBrowserAuthenticationRequest` creates the root/auth session, resolves the `browser` flow, builds an `AuthenticationProcessor`.
2. `processor.authenticate()` → `DefaultAuthenticationFlow.processFlow()` walks executions (SSO cookie check, Kerberos, username/password form...) — typically returns a challenge (renders the login form).
3. Browser POSTs to `LoginActionsService.authenticateForm` → new processor rebuilt from the auth session → `authenticationAction(execution)` → `UsernamePasswordForm.action()` → `validateUserAndPassword` (brute-force check + `credentialManager().isValid`) → `context.success()`.
4. Flow continues into any configured OTP/WebAuthn/conditional executions.
5. `authenticationComplete()` — required actions evaluated/executed (looping through `LoginActionsService.requiredActionGET/POST` if any are pending).
6. `finishAuthentication()` creates the `UserSessionModel`/`AuthenticatedClientSessionModel`, then `AuthenticationManager.redirectAfterSuccessfulFlow()` hands off to the OIDC or SAML `LoginProtocol`, which redirects the browser back to the client.
7. The client later calls the token endpoint to exchange the code for tokens (chapter 5).

Continue to
[04-identity-brokering-and-federation.md](04-identity-brokering-and-federation.md)
to see how step 2 changes when the configured authenticator is "redirect to
an external IdP" or "look this user up in LDAP" instead of a local password
form.
