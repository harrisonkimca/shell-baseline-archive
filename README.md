# Shell

## Purpose

This document defines the long-term vision and architectural direction of
the project.

Responsibilities

* Describe the project's purpose, quality bar and intended capabilities
* Define the architectural philosophy
* Describe the intended technology stack and the deployment decisions it
  leaves open
* Describe the long-term repository structure
* Communicate the project's guiding principles

This document intentionally changes very rarely.

It does not track implementation progress or the current state of the
repository. `docs/roadmap.md` owns the implementation plan.

## Project Overview

An enterprise-grade, production-ready passwordless authentication and
database foundation that new projects are forked from.

Shell is a starter, not a shared service: each new project forks it and adds
its own bounded contexts beside the identity context Shell provides. Every
fork serves web, mobile and agent clients from the same backend, and
project-specific values (name, domains, issuers, client IDs) live in
configuration, so starting a fork changes settings, not code.

Production-ready means a secure, correct and operable first release,
measured against:

* OWASP ASVS Level 2, with single-factor subscriber sign-in as a documented
  relaxation (`docs/spec/auth.md`);
* NIST SP 800-63B-4: staff sign in with passkeys, and staff sessions follow
  AAL2 reauthentication limits. Subscribers sign in with email codes or
  Google, for which Shell claims no NIST level, so every subscriber account
  is only as strong as its mailbox or Google account; subscriber sessions
  follow AAL1 limits;
* privacy law as a design constraint: Korea's PIPA, and GDPR if a project
  serves EU users.

Formal certifications such as SOC 2 or ISO 27001 are not part of the
baseline (see Planned Evolution).

Authentication itself is delegated to an identity provider, self-hosted Ory
Kratos and Ory Hydra by default. Every self-hosted component can be replaced
by a managed service without redesign; whether to, and which, is a
deployment decision made whenever a fork is deployed (see Deployment
Decisions). Subscribers and staff live in two separate identity realms.
Shell owns everything around them: the backend-for-frontend, token
verification, users, staff, roles, administration, the audit log and
business data.

The project is intentionally architected so it can evolve from a modular
monolith into an event-driven system without requiring fundamental redesign.
That evolution is planned, not pre-built: the baseline contains only what
the first production release needs (see Planned Evolution).

The goal is not merely to build authentication, but to establish an
architecture capable of supporting web applications, mobile applications, AI
agents, and future distributed services while remaining understandable for a
single maintainer.

## Vision

Build a professional-grade authentication platform that provides:

* Passwordless authentication, with no passwords anywhere
* Optional passkeys (WebAuthn) that confirm it's you on a trusted device
  instead of signing in again; never a sign-in method for subscribers
* Email codes for sign-up and sign-in, which is also the way back in,
  without leaving the current window
* Google sign-in
* Trusted devices: an optional switch that limits risky actions to devices
  the user has enrolled
* Separate directories for subscribers and staff
* Web, mobile and agent clients, with scopes and audiences
* Short-lived access tokens, verified locally on every request
* Refresh-token rotation, a list of signed-in devices, and signing out one
  device or all
* Self-service account management: passkeys, Google link, trusted devices,
  profile, sessions and deletion
* Administration of users and sessions
* An audit trail of security-relevant actions
* Internationalization: languages and timezones added as data, not code

The platform favors long-term maintainability over short-term convenience.

## Architecture Philosophy

The architecture is guided by a small number of long-lived principles.

### Modular Monolith First

The system begins as a modular monolith.

This provides clear module boundaries while avoiding the operational
complexity of distributed systems.

Individual bounded contexts should be capable of later extraction into
independent services without requiring significant redesign.

Microservices are considered an evolutionary outcome — not an initial
architectural choice.

Our application is one stateless deployable that scales horizontally;
shared state lives in PostgreSQL and Redis. The identity providers run
beside it and are not part of the monolith.

### Domain-Driven Design

Business capabilities are organized into bounded contexts with domain models
that remain independent of frameworks, persistence, and transport concerns.

Each context follows layered architecture:

* Domain
* Application
* Infrastructure
* Presentation

Dependencies point inward: the domain depends on nothing outside itself,
the application layer depends on the domain, and infrastructure and
presentation depend on both. Domain and application code never import a
framework, SDK or database driver.

The baseline starts with one bounded context, identity. A new context is
added when a capability has its own model and rules, not for every feature;
forks add their own business contexts this way. Contexts talk only through
each other's application interfaces, never through each other's internals
or tables. These rules are checked in CI, not left to convention.

### Authentication Philosophy

Authentication is passwordless by design and delegated. The identity
provider holds identities, credentials and sign-in flows, and issues OAuth
2.0 and OpenID Connect tokens. Shell verifies tokens; it never issues them.

By default the provider is self-hosted Ory: Kratos for identities and
flows, Hydra for tokens, with Kratos as Hydra's login provider. Both run in
production mode in every environment, so relaxed dev-mode checks never hide
problems until deployment. A fork may replace Ory with a managed provider at
deployment (see Deployment Decisions).

Sign-in methods at launch:

* Subscribers: email codes and Google
* Staff: passkeys (WebAuthn), with user verification required

A subscriber's optional passkeys, which also require user verification,
confirm it's them on a trusted device instead of signing in again; they
never sign anyone in.

Passwords are disabled. Every email step (sign-up, sign-in and confirmation)
uses a one-time code entered in the current window, never a link.

Every subscriber account and session is treated the same, with no separate
recovery path and no address change. Signing in again with an email code is
also how a user gets back in, so every account is only as strong as its
mailbox or Google account (`docs/architecture.md`). Changes that affect how
someone signs in, logging out other devices and deleting the account need a
fresh sign-in. Accounts are never linked automatically on a matching email.

An optional switch, Require trusted devices, limits risky actions to devices
the user has enrolled and asks for a recent sign-in before them, so a copied
session can't use them; a passkey never gates a user's control of their own
account. A fork may force the switch on for every account when it deems its
risky actions important, such as ones involving money or sensitive data.
Notifications report sign-ins from new devices and every change to sign-in
methods, trusted devices and deletion. Exact rules and values are in
`docs/spec/auth.md`.

```
  Subscribers                                  Staff
  web app · mobile app · agents                React-admin console
          │                                            │
          ▼                                            ▼
  HTTPS edge: TLS, rate limits, security headers
          │                                            │
          ▼                                            ▼
  FastAPI: BFF and resource server ──────────► PostgreSQL · Redis
    customer routes accept customer-realm tokens only
    admin routes accept workforce-realm tokens only
          │                                            │
          ▼                                            ▼
  Customer realm                               Workforce realm
  email codes · Google                         passkeys only
```

Sign-in screens reach the identity provider's public endpoints only through
the HTTPS edge, and the backend never depends on which sign-in UI is in
use. Each realm is its own Kratos and Hydra pair, or its own
tenant if a managed provider is chosen.

FastAPI is the backend-for-frontend for browsers and the resource server for
every client. For browsers it runs the authorization-code flow with PKCE
through a protocol-level OAuth client, never hand-written protocol code, and
keeps the access and refresh tokens in cookies that browser JavaScript
cannot read. Mobile and agent clients send `Authorization: Bearer` tokens
instead; CSRF protection applies to the cookie path only. Every request is
verified locally against the issuer's cached public keys, with no call to
the provider; if the keys are unavailable, requests are refused. A copied
access token stays valid until it expires or its device is logged out, an
accepted cost of self-contained tokens.

Exact behavior is defined in `docs/spec/auth.md`. The token flow, data
placement, and what must still be verified before launch are in
`docs/architecture.md`.

### Identity Realms

Subscribers (customer identity) and staff (workforce identity) live in two
separate realms, each with its own issuer, signing keys, audience, cookies
and host. The separation is enforced by who issued a token, never by a role
flag that a subscriber account could acquire:

* **Customer realm:** subscribers; email codes and Google;
  self-registration; passkeys only to confirm.
* **Workforce realm:** staff; passkeys only; no self-registration, no email
  codes and no self-service recovery. Only an operator adds, re-enrolls or
  removes staff, through a script, and every change is announced to all
  admins.

Admin routes accept only workforce-realm tokens with the admin audience;
subscriber routes accept only customer-realm tokens. No route accepts both,
so a subscriber token is rejected before any role check runs.

Neither realm grants access to cloud infrastructure. Cloud console access
goes through the cloud provider's own identity and access management, with
MFA, and the app runs with a least-privilege workload identity. The admin
API has no operation that changes infrastructure.

### Clients, Scopes and Audiences

Every client is registered with its realm's provider as configuration:

* **Web:** our BFF, a confidential client.
* **Mobile:** a public client using PKCE, signing in through the system
  browser.
* **Agents acting as themselves:** client credentials.
* **Agents acting for a user:** the same PKCE sign-in as mobile, with
  narrower scopes; agents can't pass a check that needs a sign-in.

Each API has one audience. Scopes are named `resource:action`, such as
`profile:read`, and declared in code; every route states the scopes it
needs, and anything undeclared is denied. Each route also declares its gate:
a fresh sign-in whatever the Require trusted devices switch says, a trusted
device while the switch is on, both, or neither, chosen for each action as
it is built. Token verification turns any token into one principal (user or
service, realm, subject, scopes, audience), and the rest of the app sees
only that principal.

The API is versioned in its URL (`/v1`), changes only additively within a 
published version, keeps superseded versions for a published deprecation 
window, and publishes OpenAPI as its contract, because mobile clients 
cannot be forced to update. Client details are in `docs/architecture.md`.

### State Machine Driven Workflows

Authentication flow state lives in the identity provider, not
in our code. Shell keeps an explicit state machine only for its own
multi-step workflows, and only where one is needed. Persistent lifecycle
state, such as whether a user is disabled, is distinct from workflow
progression and stays on the entity it concerns.

The likely first case is disabling or deleting a user, which is coordinated
across the identity provider and our database: several steps, each retried,
with partial failures recorded. A state-machine library joins the stack only
when such a workflow needs one.

### Identity Model

Credentials (passkeys, email addresses, linked Google identities) live in
the identity provider; Shell keeps no copy and looks an address up through
the user's ID when it needs one. The one exception is trusted devices: Shell
holds their public keys itself, because the provider has no such feature.
Subscribers are rows in `users` and staff are rows in `staff`, each keyed by
our own UUID. Every table, Shell's and every fork's, keys a user by that
UUID and never stores the address. Each context migrates only its own
tables, and schema changes add before they remove, so running code never
meets a schema it can't read. When an account is deleted, every row a fork
keeps for that user is deleted or cleared with it. The provider's identity
ID (unique) and its issuer are plain columns, and a token's subject is
mapped to our ID. Roles reference `staff` only, no table links the two
realms, and the audit log records which realm each actor belongs to. New
sign-in methods are added as provider configuration, not as new aggregates.
The provider's databases are never read or written directly.

### Ports and Adapters

Every external system sits behind a port: a contract owned by the domain or
application layer and implemented by an adapter in infrastructure.
Repositories are the ports for our database; `TokenVerifier`,
`OidcClient` and `IdentityAdmin` are the ports for the identity provider;
`EmailSender`, Redis and any future provider follow the same rule. The
application depends on contracts rather than technology, so any component
can be replaced by writing a new adapter. A port with more than one
plausible implementation, such as the identity-provider ports, carries a
contract-test suite that every adapter must pass.

This is what lets a fork swap any self-hosted system for a managed one with
minimal redesign:

* Everything vendor-specific lives in an adapter or in configuration:
  issuer and key URLs, admin endpoints and credentials, and the mapping of
  each vendor's claims onto our principal.
* The spec states the capabilities any identity provider must offer, not
  Ory features.
* The sign-in screen is the most vendor-specific frontend code, so it sits
  in its own module in `web/`.
* PostgreSQL and Redis use only features their managed versions support,
  connections require TLS, and nothing needs superuser rights.

A first deployment starts empty: development data is synthetic and is never
migrated.

### Unit of Work

Application use cases execute inside a Unit of Work. The Unit of Work
represents the transactional boundary of the application. Persistent changes
made within that boundary commit atomically, preventing partial repository
writes when a later operation fails.

The Unit of Work covers our database only. Operations that span our database
and the identity provider, such as disabling a user, are coordinated by a
use case that retries partial failures and records them in the audit log.

### Administration

Administration is a React-admin console, a separate app in `apps/admin/` on
its own host, so its code never ships to subscribers and the host can later
be restricted at the network level. Staff sign in through the workforce
realm. The console calls FastAPI admin endpoints only; those call use cases,
which reach app data through repositories and subscriber identity actions
through `IdentityAdmin`. Every admin endpoint requires a workforce-realm
token, the admin role and CSRF protection, and writes an audit entry;
destructive actions also need a confirmation step, and high-risk ones a
fresh passkey confirmation, within a window set in `docs/spec/auth.md`.
Revocation always covers both sessions and refresh grants. The identity
provider's admin APIs are never published. The console also has a Data
section, where each table a context registers gets list, search, add, edit
and delete pages through the same admin endpoints; schemas change only
through migrations. Capabilities are listed in `docs/architecture.md`.

### Error Handling

The domain raises domain errors, and adapters translate provider and
database errors into them, so no provider or SQL error leaks past an
adapter. One FastAPI handler maps errors to RFC 9457 Problem Details with
stable error codes and no internals. Shell's authentication errors stay
generic, so they never reveal whether an account exists; the provider's
code sign-in can, which is an accepted risk. Every request carries a
correlation ID that is passed to the provider and written to structured
logs. Provider calls have timeouts and limited retries; if the provider is
down, existing sessions keep working until their tokens expire and new
sign-ins fail with a clear error. Details are in `docs/architecture.md`.

### Internationalization

Languages and timezones are added as data, never as code changes.
Timestamps are stored and sent in UTC and shown in each user's own
timezone, and every user-visible string lives in a message catalogue from
day one. The backend never translates interface text; clients translate
errors by their stable codes. Emails are the exception, rendered in the
recipient's language by our email sender. The first release ships English
and Korean, so two languages prove the setup from the start. Details are in
`docs/architecture.md`.

### Operations

The baseline runs in two environments, development and production, built
from the same configuration-as-code so they differ as little as possible:

* **Development:** Docker Compose running the self-hosted defaults from
  Deployment Decisions, with local certificates, secrets in a git-ignored
  `.env`, email captured by Mailpit, and synthetic data only.
* **Production:** the same components, each kept self-hosted or replaced by
  a managed service as described in Deployment Decisions, with every
  connection encrypted, internal ones included.

Identity-provider email goes through our `EmailSender`, so per-address rate
limits, templates and the email provider are ours to change. Both
environments share CI/CD that applies migrations, health and readiness
probes, structured logs with correlation IDs, and basic metrics. All
databases, whether self-hosted or managed, are backed up automatically, and
at least one restore is tested.
Images are pinned by digest and scanned for vulnerabilities. ASVS Level 2
controls are written as tests as features are built, and every behavior
Shell relies on from the identity provider is proven against the real
provider in CI, not taken from its documentation. Account deletion
reaches both the identity provider and our database. Details, including
signing-key rotation, are in `docs/architecture.md`.

### Planned Evolution

The architecture must permit events, CQRS and distribution without making
any of them a baseline requirement. Complexity is added when its trigger
appears, not before. Event-driven patterns are first established and
validated within the modular monolith before external messaging or service
extraction is introduced.

The baseline already contains the seams that make each later step an
addition rather than a redesign:

| Seam in the baseline | What it makes cheap later |
|---|---|
| Use cases as the single entry point to each context | Publishing domain events; splitting commands from queries (CQRS) |
| Unit of Work around every use case | An outbox written in the same transaction, for reliable event publication |
| Ports and adapters with contract tests | Replacing the identity provider, database or any other component |
| Realms separated by issuer | Changing either realm's provider without touching the other |
| Every table keys users by our UUID and never stores the address | Changing a subscriber's address at the provider alone, if it is ever needed |
| One principal type with scopes and audiences | New client types and third-party access |
| Gates declared per route | Gating a fork's new actions, mandatory or switch-dependent, without redesign |
| Versioned API with an OpenAPI contract | Evolving the API without breaking older mobile apps |
| Context boundaries enforced in CI | Extracting a context as an independent service |
| Message catalogues and UTC timestamps | New languages and timezones as data |
| Stateless app, shared state in PostgreSQL and Redis | Horizontal scaling and, later, multi-region |

| Later | Trigger |
|---|---|
| Domain events, CQRS, sagas | A real need appears |
| Service extraction, multi-region | A real need appears |
| Apple sign-in | iOS users or product demand (one social-login entry in the provider and an Apple developer account; no app-code change) |
| Opt-in advanced protection: sign-in needs the email code and a passkey, with recovery codes | A fork serves people at higher risk of targeted attacks, or holds money or sensitive data |
| Third-party clients and a consent screen | A partner or third-party app needs access |
| Token exchange, device authorization flow | An agent, CLI or TV client needs them |
| Dynamic client registration | Many external clients |
| Changing a subscriber's address, by an operator after a check that doesn't rely on the mailbox | Users must keep their data after losing a mailbox, or a customer's email domain changes |
| More admin roles | Support staff join |
| Staff sign-in through a fork's own identity provider, and managed devices for the console | A fork with a company identity provider, or ASVS Level 3 |
| Staging environment | Regular releases or a team |
| Tracing and SLO alerts | Several services or on-call |
| RPO/RTO targets and disaster-recovery drills | Paying customers |
| SBOM | A customer or compliance request |
| SOC 2 or ISO 27001 certification | An enterprise customer requires it |
| mTLS between services | Shared infrastructure or a compliance demand |
| Data export, incident runbooks | GDPR scope or a team |
| A command that generates a new feature's skeleton (context, migration, registration) | Forks add features often |

`docs/roadmap.md` owns the implementation sequence.

## Technology Direction

The technologies below define the intended project stack; substitutions require a deliberate change to the project's architectural direction.

| Technology | Purpose | Note |
|---|---|---|
| Python 3.13 | Language and runtime | Pinned in `.python-version` and `pyproject.toml`; installed by uv |
| FastAPI | Backend-for-frontend and resource server | Publishes the OpenAPI contract |
| Pydantic v2 | Validation and data models | |
| pydantic-settings | Application settings | Same code reads `.env` or the secrets manager |
| SQLAlchemy 2.x | ORM | |
| PostgreSQL | Primary database | Self-hosted by default; a managed service is a deployment decision |
| Redis | Rate limiting, refresh serialization, job queue, single-use gate challenges and continuation codes | Self-hosted by default; a managed service is a deployment decision |
| asyncpg | Async PostgreSQL driver | |
| redis-py | Async Redis client | |
| arq | Background job queue | |
| PyJWT | Local access-token verification | Verification only; the provider issues tokens |
| Authlib | OAuth 2.0 client for the browser code flow with PKCE | Low-level `AsyncOAuth2Client` only, not a framework |
| HTTPX | HTTP client for identity-provider admin APIs | |
| Jinja2, Babel | Localized email templates | Rendered by `EmailSender` |
| maxminddb | Reads the DB-IP City Lite file | MaxMind's MMDB reader; lookups stay on the server |

Identity provider

| Technology | Purpose | Note |
|---|---|---|
| Ory Kratos | Identities, credentials, sign-in flows | Default; one per realm, production mode |
| Ory Hydra | OAuth 2.0/OIDC tokens, signing keys, JWKS | Default; one per realm, production mode |
| Ory reference UI | Sign-in, sign-up, sign-out and settings screens | Development only; replaced by our own screen before launch. Our app runs the consent step in every environment |
| Managed identity provider | Optional replacement for Ory | A deployment decision, scored against the capability checklist |

Frontend

| Technology | Purpose | Note |
|---|---|---|
| Node.js (current LTS) | Frontend build and tooling | Pinned in `.nvmrc` |
| React | UI framework | |
| TypeScript | Static typing | API client generated from OpenAPI |
| React Router | Client-side routing | |
| FormatJS (`react-intl`) | Message catalogues, ICU plurals, formatting | |
| React-admin | Administration console, including the Data section | Own app and host; its own translation provider |
| Swift, Kotlin | Native mobile test clients | XCTest on iOS, Android instrumented tests; test-only |

Infrastructure

| Technology | Purpose | Note |
|---|---|---|
| Docker | Containerization | Images pinned by digest |
| Docker Compose | Local orchestration | Also the default deployment platform |
| Caddy | TLS proxy and edge rate limits | Built with a rate-limit module; a cloud edge may replace it at deployment |
| Mailpit | Email capture | Development only |
| Google Workspace (SMTP relay) | Production email | Interim; dedicated provider later |
| Secrets manager | Secrets in deployed environments | A deployment decision |
| DB-IP City Lite | IP-to-city data for the device list | Monthly download; CC BY 4.0, so pages showing a location link "IP Geolocation by DB-IP" |

Tooling

| Technology | Purpose | Note |
|---|---|---|
| uv | Package/environment management | |
| Ruff | Linting and formatting | |
| Pyright | Static type checking | Strict mode |
| import-linter | Layer and context boundary checks | Runs in pre-commit and CI |
| Alembic | Database migrations | Our database only; run as a CI job |
| pytest | Testing | |
| pre-commit | Git hook framework | |
| GitHub Actions | Continuous integration | |

## Deployment Decisions

Shell is deployment-ready: any fork can be deployed to any region, with any
component kept self-hosted or replaced by a managed service, without
changing domain or application code. A swap is either configuration or a
new adapter that passes the port's contract tests. These choices are made
when a fork is deployed, not while Shell is designed, and can be revisited
for each new deployment. Forking Shell does not trigger them: a fork keeps
developing on the defaults below for as long as it needs, and makes these
decisions only when it is first deployed.

Shell sets no default region, and any deployed environment refuses to start
without one. Privacy law follows the fork's operator and users, not the
region, so whichever region is chosen is stated in the fork's privacy
policy, and data stored outside its users' country is checked against that
country's cross-border rules.

| Decision | Default | Any choice must | Plugs in through |
|---|---|---|---|
| Region | None; required | Hold every component in the same region | Configuration |
| Sign-up terms | None; required (placeholders in development only) | Include the operator's own terms, privacy policy and required confirmations, such as a minimum age | Configuration |
| Compute platform | Docker Compose on one VM | Run the image with health probes, a private network whose connections are encrypted, and at least two `api` replicas | Configuration |
| Identity provider, per realm | Self-hosted Ory Kratos and Hydra | Pass the capability checklist, including region availability; then chosen on cost | Configuration for Ory Network; otherwise new adapters and sign-in module |
| PostgreSQL | Self-hosted | Support the features we use, require TLS, need no superuser | Configuration |
| Redis | Self-hosted | Require TLS and authentication | Configuration |
| Secrets manager | None; required (`.env` in development only) | Be the only source of secrets outside development | Settings source |
| Edge | Caddy with a rate-limit module | Terminate TLS, apply rate limits and security headers | Configuration |
| Email | Mailpit (development); Google Workspace relay | Send from our domain with SPF, DKIM and DMARC | `EmailSender` adapter |
| Audit storage | None; required at deployment | Preserve deletion audit records independently of database restores, with durable writes and append-only application access | Adapter |

How each component plugs in is described in `docs/architecture.md`.

## Repository Layout

The repository is organized by architectural responsibility rather than
technical framework.

```
apps/
    api/
        src/
            shell_api/
                contexts/
                shared/
                core/
    web/
    admin/

docs/

scripts/

tests/

docker/
```

The exact contents of each directory may evolve over time while preserving
the overall architectural organization.

`apps/` contains the application source code. `contexts/` holds business
capabilities organized as bounded contexts, each owning its own domain,
application, infrastructure, and presentation layers; the identity context's
identity-provider adapters are the only code that calls the provider.
`shared/` holds abstractions intentionally reusable across bounded contexts.
`core/` holds cross-cutting application configuration and bootstrapping.
`web/` holds the React frontend, with the sign-in screen in its own module.
`admin/` holds the React-admin console. Both call the backend's API, and
both are distinct from each bounded context's own backend-side presentation
layer above, despite the shared name. Production mobile apps live in the projects 
that fork Shell. Shell includes a reference mobile test client in `tests/mobile/`.

`docs/` holds project documentation: vision, architecture, specification,
roadmap, decisions and guides.

`scripts/` holds repository tooling that CI runs, such as the quality-gate
script, and the staff provisioning script. `tests/` mirrors the application
architecture. `docker/` holds local development infrastructure, including
Caddy and Ory configuration.

## Documentation

| Document | Purpose |
|---|---|
| `README.md` | Project vision and architecture principles |
| `docs/architecture.md` | Realms, clients, token flow, data placement, administration, error handling, internationalization, operations, how deployment decisions plug in, threat model |
| `docs/spec/auth.md` | Authentication and account requirements, each with a stable ID |
| `docs/roadmap.md` | Implementation roadmap |
| `docs/decisions.md` | Durable architectural decisions |
| `docs/guides/` | How-to guides for forks, starting with adding a feature: its context, tables, migrations, routes and Data-section registration, and its clean-up on account deletion |

## Development Philosophy

Project values:

* Maintainability over cleverness
* Explicit contracts over implicit behavior
* Strong typing
* Simple abstractions
* Clear boundaries
* Incremental evolution
* Documentation-driven development

Repository principles:

* Every implementation should reduce ambiguity
* Every abstraction should have a clear responsibility
* Every dependency should justify its long-term maintenance cost
* Every workflow should preserve architectural boundaries
* Every change should improve implementation, documentation,
  consistency, and maintainability

## Open Questions

None. Region, sign-up terms, compute platform, managed services, secrets
manager, edge, email provider and independent audit storage are deployment
decisions, listed in Deployment Decisions above.
