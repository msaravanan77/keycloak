# 6. Authorization Services (Fine-Grained Authorization / UMA2)

Realm and client **roles** (RBAC) answer "what is this user allowed to do
in general." Authorization Services is a separate, optional layer on top
that answers a finer question: "is this specific subject allowed this
specific scope on this specific resource, right now, according to a set of
composable policies" — and can hand back that decision either as a boolean,
a permission list, or a signed **RPT** (Requesting Party Token, itself just
an access token with an embedded permission list).

## 6.1 Core model

`server-spi-private/src/main/java/org/keycloak/authorization/model/`:

| Interface | Represents |
|---|---|
| `ResourceServer` | A client acting as an authz-protected API — enforcement mode, decision strategy, `getClientId()`. |
| `Resource` | A protected asset — owner, URIs, scopes, type. |
| `Scope` | An action/permission on a resource (e.g. `view`, `edit`). |
| `Policy` | An access-control rule — `type`, `DecisionStrategy`, `Logic`, a config map, and associated policies/resources/scopes (policies can compose other policies). |
| `PermissionTicket` | A UMA2 grant record linking resource/scope/requester/owner/policy plus granted state. |

Persistence contracts, `server-spi-private/.../authorization/store/`:
`ResourceStore`, `ScopeStore`, `PolicyStore`, `PermissionTicketStore`,
`ResourceServerStore`, `StoreFactory`, `AuthorizationStoreFactory`,
`StoreFactorySpi`. The session-scoped facade tying it together is
**`AuthorizationProvider`** (`server-spi-private/.../authorization/AuthorizationProvider.java`)
— exposes `StoreFactory`, policy providers, and evaluators.

## 6.2 Built-in policy types

`authz/policy/common/src/main/java/org/keycloak/authorization/policy/provider/`:

| Policy | Class | Condition |
|---|---|---|
| Role | `role/RolePolicyProvider(+Factory)` | Realm/client role membership (`RoleModel`, optional deep composite lookup) |
| User | `user/UserPolicyProvider(+Factory)` | Specific users |
| Client | `client/ClientPolicyProvider(+Factory)` | Specific OIDC clients (service accounts) |
| Client Scope | `clientscope/ClientScopePolicyProvider(+Factory)` | Granted client scopes |
| Group | `group/GroupPolicyProvider(+Factory)` | Group membership |
| Time | `time/TimePolicyProvider(+Factory)` | Date/time windows |
| Regex | `regex/RegexPolicyProvider(+Factory)` | Regex match against a claim |
| JS/Rule | `js/JSPolicyProvider(+Factory)`, `DeployedScriptPolicyFactory` | Deployed JS scripting logic |
| Aggregated | `aggregated/AggregatePolicyProvider(+Factory)` | Composes other policies (AND/OR via `DecisionStrategy`/`Logic`) |
| (Permission objects) | `permission/{Resource,Scope,UMA}PolicyProvider(+Factory)` | Not conditions per se — these represent the "permission" itself, referencing the policies above |

All implement **`org.keycloak.authorization.policy.provider.PolicyProvider`**.
`RolePolicyProvider` additionally implements `PartialEvaluationPolicyProvider`
(`org.keycloak.authorization.fgap.evaluation.partial`) — used by the newer
**Fine-Grained Admin Permissions (FGAP)** feature, which reuses this exact
engine to authorize the Admin REST API itself (see §6.6 and chapter 8).

## 6.3 The evaluation engine

`server-spi-private/src/main/java/org/keycloak/authorization/policy/evaluation/`:

- **`PolicyEvaluator` / `DefaultPolicyEvaluator`** — `evaluate(ResourcePermission, AuthorizationProvider, EvaluationContext, Decision, decisionCache)`. If enforcement mode is `DISABLED`, or the permission is already granted, auto-grant. Otherwise, looks up applicable `Policy` rows from `PolicyStore` (by resource, resource type, or scopes) and invokes each policy's `PolicyProvider.evaluate(DefaultEvaluation)`. If nothing matched and mode is `PERMISSIVE`, auto-grant (unless the resource is owner-managed/UMA).
- **`DefaultEvaluation`** (`implements Evaluation`) — per-policy evaluation context, walks aggregated-policy trees.
- **`EvaluationContext`** — request-time identity/attribute context (`DefaultEvaluationContext` in `services/.../authorization/common/`).
- **`Decision`** — accumulates PERMIT/DENY per permission.
- **`AbstractDecisionCollector`, `DecisionPermissionCollector`, `DecisionResultCollector`, `PermissionTicketAwareDecisionResultCollector`** — shape the final output (grouped permissions / boolean decision / detailed result), depending on requested `response_mode`.
- **`Permissions` / `ResourcePermission`** — the "requested permission" unit (resource + scopes) fed into the evaluator.

## 6.4 UMA2 endpoints and the token-endpoint grant

Grant wiring: `OAuth2Constants.UMA_GRANT_TYPE =
"urn:ietf:params:oauth:grant-type:uma-ticket"`, registered via
`PermissionGrantTypeFactory` and handled by
`PermissionGrantType extends OAuth2GrantTypeBase`
(`services/.../protocol/oidc/grants/PermissionGrantType.java`) — invoked
from `TokenEndpoint` exactly like any other grant type (chapter 5).
`PermissionGrantType.process()` extracts the bearer token/claims and
delegates to **`AuthorizationTokenService`**
(`services/.../authorization/authorization/AuthorizationTokenService.java`)
— the RPT-issuing engine: parses the `AuthorizationRequest`
(permissions/ticket/claim tokens), resolves `ResourcePermission`s, runs
`DefaultPolicyEvaluator`, and builds an `AccessTokenResponse` (RPT) via
`TokenManager`, embedding `AccessToken.Authorization` (the granted
`Permission` list) into the token — or, per `response_mode`, returns a
`decision` boolean or a raw permission list instead.

The **Protection API** (UMA "resource registration"/permission-ticket
management), mounted at `/protection` under
`org.keycloak.authorization.AuthorizationService`:

| Path | Class | Purpose |
|---|---|---|
| `/resource_set` | `resource/ResourceService` | UMA Resource Registration API |
| `/permission`, `/permission/ticket` | `permission/PermissionService`, `permission/PermissionTicketService` | Issue/manage permission tickets |
| `/uma-policy` | `policy/UserManagedPermissionService` | User-managed policies over owned resources |
| `/token/introspect` | `introspect/RPTIntrospectionProvider(+Factory)` | RPT introspection |

`UmaConfiguration.java` publishes the UMA2 discovery document.

## 6.5 Client-side policy enforcement

Only the **configuration contract** lives in this repo:
`core/src/main/java/org/keycloak/representations/adapters/config/PolicyEnforcerConfig.java`
(protected paths, enforcement mode, lazy-load-paths, claim-information-point
config, HTTP-method-to-scope mapping). The actual enforcer runtime that
adapters embed (matching incoming requests to `PathConfig`s and calling the
Protection/Token API above) lives outside this monorepo slice — this repo
only defines the shape adapters agree on.

## 6.6 Admin management REST

`services/src/main/java/org/keycloak/authorization/admin/`:

- `AuthorizationService` — mounted via `ClientResource.authorization()` at `/admin/realms/{realm}/clients/{id}/authz`, exposes `/resource-server` → `ResourceServerService`.
- `ResourceServerService` — CRUD on the resource server (create/update/delete/find, `/settings` export, `/import`); mounts `/resource`→`ResourceSetService`, `/scope`→`ScopeService`, `/policy`→`PolicyService`, `/permission`→`PermissionService`.
- `PolicyService`/`PolicyResourceService`/`PolicyTypeService`/`PolicyTypeResourceService` — CRUD plus per-type config discovery.
- `PolicyEvaluationService` (+ `PolicyEvaluationResponseBuilder`/`FGAPPolicyEvaluationResponseBuilder`) — powers the Admin Console's "Evaluate" tab: a dry-run of the policy engine against a simulated identity/resource/scope.

All guarded by `AdminPermissionEvaluator` (chapter 8).

## 6.7 Relationship to RBAC

Authorization Services sits **on top of**, not instead of, standard
realm/client roles: `RolePolicyProvider` directly queries
`RealmModel.getRoleById()` / `identity.hasRealmRole()` /
`identity.hasClientRole()` (including deep composite-role resolution via
`RoleUtils.getDeepUserRoleMappings`). So a "Role" policy condition is
satisfied by ordinary RBAC role assignments — RBAC roles become one of
several possible policy *conditions* (alongside user/client/group/time/
regex/js/aggregated), which are combined into policies, attached to
resource/scope permissions, and evaluated per request to produce a decision
or an RPT.

The newer **Fine-Grained Admin Permissions (FGAP)** feature
(`org.keycloak.authorization.fgap.*`,
`services.resources.admin.fgap.*`) reuses this identical policy/evaluation
engine to authorize the **Admin REST API itself** — a hidden per-realm
"admin permissions" client acts as the resource server, and
`RolePolicyProvider implements PartialEvaluationPolicyProvider` so admin
RBAC can itself be expressed and evaluated through this engine. See chapter
8 for how `AdminPermissionEvaluator` picks between classic role-based
(`MgmtPermissions`) and FGAP (`MgmtPermissionsV2`) checking.

## 6.8 Evaluation sequence (UMA2 RPT flow)

```
1. Client obtains a permission ticket from /protection/permission
   (or already knows the resource/scope names to request directly).

2. Client → POST /realms/{realm}/protocol/openid-connect/token
     grant_type=urn:ietf:params:oauth:grant-type:uma-ticket
     Authorization: Bearer <access_token>
     ticket=... / permission=... / claim_token=...
   → TokenEndpoint routes to PermissionGrantType.process().

3. PermissionGrantType builds a KeycloakAuthorizationRequest and
   delegates to AuthorizationTokenService.

4. AuthorizationTokenService resolves the identity (KeycloakIdentity),
   loads requested resources/scopes via ResourceStore/ScopeStore,
   builds ResourcePermission objects (Permissions helper).

5. For each ResourcePermission, DefaultPolicyEvaluator.evaluate() fetches
   associated Policy rows and invokes each PolicyProvider.evaluate(...)
   (role/user/client/group/time/js/regex/aggregated — aggregated policies
   recurse into their children, applying DecisionStrategy/Logic).

6. Decision/DecisionResultCollector accumulates PERMIT/DENY per
   permission, honoring the resource server's overall DecisionStrategy
   (AFFIRMATIVE/UNANIMOUS/CONSENSUS) and PolicyEnforcementMode
   (ENFORCING/PERMISSIVE/DISABLED).

7. Granted permissions become Permission objects embedded into a new
   access token via TokenManager → returned as the RPT
   (AccessTokenResponse) — or, per response_mode, a decision boolean or
   raw permission list instead.

8. The resource server later introspects the RPT at
   /protection/token/introspect (RPTIntrospectionProvider) to authorize
   the actual protected-resource request.
```

Continue to
[07-domain-model-and-persistence.md](07-domain-model-and-persistence.md) —
every `RealmModel`/`UserModel`/`Resource`/`Policy` object mentioned above
ultimately has to be stored and cached *somewhere*; this is that chapter.
