# Keycloak — Deep Architecture Knowledge Base

This folder is a from-source, code-verified deep dive into the Keycloak Java
codebase. It exists so a newcomer — engineer, SRE, or security reviewer — can
go from "never seen this repo" to "I understand exactly how a request turns
into a token" without spelunking through 25+ Maven modules on their own.

Everything here was produced by reading the actual Java source in this
repository (paths and class names are real, not paraphrased), on the
`999.0.0-SNAPSHOT` / Quarkus-based architecture (i.e. the modern `quarkus/`
distribution — legacy WildFly deployment is not covered, it no longer exists
upstream). Where a class name or file path is given, you can open it directly
and cross-check.

## How to read this

Read in order if you're new to the codebase; jump directly to a chapter if
you already know the shape of Keycloak and need one subsystem.

| # | Document | What it answers |
|---|----------|------------------|
| 1 | [01-big-picture.md](01-big-picture.md) | What are all these Maven modules? What's the 30,000-ft shape of the system? |
| 2 | [02-bootstrap-and-runtime.md](02-bootstrap-and-runtime.md) | What actually runs when you type `kc.sh start`? How does an HTTP request become a `KeycloakSession`? |
| 3 | [03-authentication-engine.md](03-authentication-engine.md) | How does the login flow engine work — flows, executions, authenticators, required actions? |
| 4 | [04-identity-brokering-and-federation.md](04-identity-brokering-and-federation.md) | How do external IdPs (social/OIDC/SAML) and external user stores (LDAP/Kerberos) plug in? This is the "multiple identity sources" chapter. |
| 5 | [05-oidc-and-saml-protocols.md](05-oidc-and-saml-protocols.md) | What are the actual wire endpoints? How are tokens minted and signed? How are SAML assertions built? |
| 6 | [06-authorization-services.md](06-authorization-services.md) | How does fine-grained authorization (UMA2, resource/policy/permission, RPTs) work, and how does it relate to plain RBAC? |
| 7 | [07-domain-model-and-persistence.md](07-domain-model-and-persistence.md) | What is a Realm/Client/User modeled as in Java? How is it persisted (JPA/Hibernate) and cached (Infinispan) and clustered? |
| 8 | [08-admin-api-events-operator.md](08-admin-api-events-operator.md) | How does the Admin REST API / Account Console / event system / Kubernetes Operator fit in? |
| — | [glossary.md](glossary.md) | One-line definitions of every acronym and Keycloak-specific term used above. |

## Diagrams

| Diagram | Purpose |
|---|---|
| [diagrams/architecture-overview.svg](diagrams/architecture-overview.svg) | Full-system component diagram: Quarkus runtime, SPI provider layer, persistence/cache, and every identity source (local DB, LDAP/Kerberos federation, brokered external IdPs), plus admin/account/operator planes. |
| [diagrams/auth-sequence-multi-idp.svg](diagrams/auth-sequence-multi-idp.svg) | Sequence diagram: one `AuthenticationProcessor` engine, three different identity sources (local password+OTP, LDAP-federated user, brokered external OIDC/SAML IdP), all converging on the same session/token issuance path. |

Both are hand-authored, dependency-free SVG (no external libraries), themed
for both light and dark viewing, and safe to open directly in a browser or
embed in other docs.

## Scope and method

- **Java only, by request.** JS frontends (`js/apps/admin-ui`, `js/apps/account-ui`, adapters in other languages) are mentioned only for orientation, never analyzed in depth.
- **Entry-point-first.** Every chapter is organized around "where does execution start, and where does it go next," matching how you'd actually debug this system with a debugger attached.
- **Verified, not guessed.** Class names, file paths, and method signatures were read directly from source in this checkout (branch `claude/codebase-knowledge-architecture-g21onw`, i.e. current `main`). A handful of load-bearing claims (main entry point, root JAX-RS path, core SPI shape) were independently re-verified by direct file reads after the initial research pass.
- **Snapshot in time.** This is version `999.0.0-SNAPSHOT` (unreleased upstream `main`). Class internals move release to release; the architectural shapes described here (SPI-driven providers, flow-engine-driven auth, LoginProtocol abstraction over OIDC/SAML, Infinispan-cached JPA model) have been stable for many major versions and are unlikely to change wholesale.

## Fastest path to "I get it"

If you only read one thing: **01-big-picture.md**, then look at
**diagrams/architecture-overview.svg** side by side with it. That pair gives
you the mental map everything else in this folder hangs off of.
