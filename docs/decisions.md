# Decisions

Shell's durable architectural decisions, every one in force: choices that shape the design, would be
costly to reverse and had a real alternative. Values, procedures and Later items live in the linked
files. Each record gives the decision, why, its trade-offs, the options considered and where the
details are. A fork adds a record under the next free ID; a record that replaces another names it in
its Decision, and no ID is ever reused.

## Foundations

### DEC-001 Shell is a starter that each project forks

**Decision.** Shell is a starter, not a shared service: each project forks it, adds its own bounded
contexts beside identity and sets its project values (name, domains, issuers, client IDs) in
configuration.

**Why.** Each project keeps its own data, deployment and security boundary, and starting one changes
settings, not code.

**Trade-offs.** A fix made in Shell reaches a fork only when that fork pulls it.

**Considered.** A shared authentication service for every project: one outage or breach reaches them
all.

**See.** [README, Project Overview](../README.md#project-overview).

### DEC-002 Types and boundaries are enforced by tools

**Decision.** Pyright runs in strict mode across the repository, and import-linter enforces the layer
and context boundaries in pre-commit and CI.

**Why.** Strict typing catches defects early, and rules left to convention erode on a project with a
single maintainer.

**Trade-offs.** Every module, a fork's included, must pass strict typing and the import contracts.

**See.** [README, Domain-Driven Design](../README.md#domain-driven-design),
[architecture §14](architecture.md#14-import-rules).

## Authentication and identity

### DEC-003 Authentication is delegated to an identity provider

**Decision.** The identity provider holds identities, credentials, email addresses and sign-in flow
state, and issues every token; Shell verifies tokens and never issues them. Sign-in methods are
provider configuration, not aggregates in Shell's domain.

**Why.** Credential storage and sign-in flows are specialist, security-critical work, and Shell holds no
credential that could leak.

**Trade-offs.** Shell relies on the provider's behaviour, so each behaviour it uses is proved against
the real provider in CI; with the provider down, no one can sign in.

**Considered.** Shell's own credential model, one aggregate for every method with a state machine for
sign-in flows: duplicates the provider and keeps credentials in Shell.

**See.** [README, Authentication Philosophy](../README.md#authentication-philosophy),
[Identity Model](../README.md#identity-model);
[architecture §6](architecture.md#6-data-ownership-and-database-layout).

### DEC-004 Self-hosted Ory Kratos and Hydra are the default provider

**Decision.** Each realm runs Ory Kratos for identities and flows and Ory Hydra for tokens, with Kratos
as Hydra's login provider, self-hosted in every environment. Replacing them with a managed provider is
a per-fork deployment decision (DEC-037).

**Why.** Open source, self-hostable in any region and meets the capability checklist, so development
runs the same provider as production.

**Trade-offs.** Shell runs and upgrades the providers, and builds what Ory lacks: device keys, enforced
user verification and the resend gap.

**Considered.** A managed provider by default: ties every fork to a vendor and its regions before the
fork has chosen either.

**See.** [architecture §5](architecture.md#5-ports-and-adapters);
[spec §2](spec/auth.md#2-identity-provider-capabilities).

### DEC-005 Ory runs in production mode in every environment

**Decision.** Kratos and Hydra run in production mode everywhere, development included, with HTTPS
issuers and Secure cookies served through a local CA.

**Why.** Development behaves like production, so relaxed checks never hide a problem until deployment.

**Trade-offs.** Development needs a local CA and HTTPS hosts.

**Considered.** Dev mode in development: simpler, but hides problems until deployment.

**See.** [architecture §13](architecture.md#13-operations-baseline);
[spec §2](spec/auth.md#2-identity-provider-capabilities) (AUTH-IDP-020).

### DEC-006 Subscribers and staff live in two realms separated by issuer

**Decision.** A customer realm (subscribers, in `users`) and a workforce realm (staff, in `staff`) each
have their own provider instance or tenant, issuer, signing keys, audience, cookies and host. Each
route trusts exactly one realm, roles reference staff only and no table links the two. Cloud
infrastructure is a third plane, reached only through the cloud provider's IAM.

**Why.** A subscriber token fails on who issued it, before any role check, so no flag or profile edit
can make a subscriber an admin; either realm's provider can be swapped alone.

**Trade-offs.** Two provider pairs to run; someone who is both staff and subscriber has two accounts.

**Considered.** One realm with an admin role: a single flag or bug between any subscriber and admin
rights. One Hydra for both realms: it has one issuer and key set, so separation would rest on audience
and client settings (AUTH-IDP-002).

**See.** [README, Identity Realms](../README.md#identity-realms);
[architecture §3](architecture.md#3-identity-realms);
[spec §8](spec/auth.md#8-sessions-tokens-and-cookies) (AUTH-TOKEN-005).

### DEC-007 Subscribers sign in with an email code or Google, which is also the way back in

**Decision.** Subscribers sign up and sign in only with an email code or Google: no passwords, no
passkey sign-in and no separate recovery. Every subscriber session is treated the same, with no
assurance tiers, so every account is only as strong as its mailbox or Google account (the email
ceiling). Shell accepts this single factor at ASVS Level 2 as a documented relaxation of ASVS 6.3.3,
claims no NIST AAL for it and holds subscriber sessions to AAL1 time limits.

**Why.** Passwords add credential management and attack surface. Whoever holds the mailbox can simply
sign in, so recovery paths, holds, cooling-offs, passkey sign-in or tiers add complexity without
stopping them. Claude's own sign-in design is the benchmark.

**Trade-offs.** A mailbox or Google takeover is an account takeover, softened by notices of sign-ins from unseen devices,
the device list, the deletion delay and the support freeze.

**Considered.** Passwords: attack surface and resets. Passkey sign-in or assurance tiers: anyone who
can sign in can add a passkey. A recovery path with holds or a cooling-off: doesn't stop the mailbox
holder. A second factor at every sign-in: kept for opt-in advanced protection (Later).

**See.** [README, Authentication Philosophy](../README.md#authentication-philosophy);
[spec §1](spec/auth.md#1-conventions-and-assurance);
[architecture §11](architecture.md#11-security-controls-rate-limiting-and-threat-model) (the email
ceiling).

### DEC-008 Every email step uses a one-time code, never a link

**Decision.** Sign-up, sign-in and confirmation emails carry a one-time code the user enters in the
window they are using; no email carries a link that signs in or verifies.

**Why.** The flow stays in one window and device. A code never sits in a URL, where logs, referrers and
mail scanners can read or spend it, and users aren't taught to follow sign-in links.

**Trade-offs.** Users type or paste a code, so code entry is built to be accessible.

**Considered.** Sign-in or verification links: open in another browser or device, leak through URLs,
are spent by scanners and invite phishing.

**See.** [spec §5](spec/auth.md#5-email-codes-and-rate-limits),
[§3](spec/auth.md#3-sign-up-sign-in-and-linking) (AUTH-SIGN-012).

### DEC-009 A subscriber's address never changes, and tables key users by Shell's UUID

**Decision.** Subscribers' addresses can't be edited, by the user or by admins; a lost mailbox means a
fresh account. Every table, Shell's and every fork's, keys a user by Shell's UUID and never stores the
address, so an address change at the provider alone stays possible (Later).

**Why.** Any way to move an account to another address lets a mailbox thief keep it after the owner
recovers the mailbox; with none, recovering the mailbox always recovers the account.

**Trade-offs.** A user who loses their mailbox loses the account's data; support may cancel billing or
delete the old account.

**Considered.** Self-service or admin address change: gives a mailbox thief a lasting takeover.

**See.** [README, Identity Model](../README.md#identity-model);
[spec §7](spec/auth.md#7-account-settings-devices-and-deletion) (AUTH-ACCT-004).

### DEC-010 Passkeys are an optional second factor that confirms only where Shell checks the device

**Decision.** A subscriber's passkeys, several per account and added only from settings, never sign
anyone in. On a trusted device with the switch on (DEC-027), a passkey stands in for signing in again,
but only on routes Shell's own gate check protects; changes on the provider's settings screens, which
can't see the device key, take an email code or Google.

**Why.** Anyone who can sign in can add a passkey, so demanding one adds no lock; it is a faster way to
confirm. Only Shell's gate check can pair the passkey step with a device-key proof.

**Trade-offs.** With the switch off, a subscriber's passkeys confirm nothing.

**Considered.** Passkey sign-in: doesn't raise the email ceiling. Requiring a passkey to confirm: no
added security. Passkeys standing in on the provider's settings screens: no device proof there.

**See.** [spec §4](spec/auth.md#4-passkeys),
[§6](spec/auth.md#6-fresh-sign-ins-and-trusted-devices) (AUTH-FRESH-006, AUTH-FRESH-008);
[architecture §8](architecture.md#8-account-management-and-sign-out).

### DEC-011 Our sign-in screen requires user verification on every passkey ceremony

**Decision.** Every passkey ceremony in both realms requires user verification and accepts 
security keys. Our sign-in screen asks for verification, and the server refuses any response 
without the user-verified flag, through the provider's configuration if it can, otherwise 
through a check Shell adds on the provider's submission path, with no public route around it.

**Why.** Neither Kratos passkey method requires verification, and its `passkey` method allows device
passkeys only.

**Trade-offs.** If the provider can't enforce verification, Shell maintains a small check on its
passkey submissions.

**Considered.** Relying on our screen alone: a stolen security key with custom software could skip the
PIN, leaving staff with one factor.

**See.** [spec §4](spec/auth.md#4-passkeys) (AUTH-PASSKEY-001);
[architecture §5](architecture.md#5-ports-and-adapters).

### DEC-012 Accounts are never linked on a matching address

**Decision.** Each sign-in method finds the account by its own key: an email code by the address,
Google by the Google account's ID. A Google account not seen before can't sign up when Google hasn't
verified its address or an account already uses it; the user signs in another way and links Google in
settings, one Google account per user.

**Why.** Linking on a matching address hands an account to whoever controls a Google account with that
address, and Kratos would link at sign-in before Shell's hooks run.

**Trade-offs.** A user who signed up by email links Google from settings before using it.

**Considered.** Automatic linking on a verified matching address: a takeover path.

**See.** [spec §3](spec/auth.md#3-sign-up-sign-in-and-linking);
[architecture §8](architecture.md#8-account-management-and-sign-out).

### DEC-013 Account enumeration through the provider's code sign-in is accepted

**Decision.** Shell's screens and errors answer alike whether or not an address has an account: one
email step sends a code either way, for sign-up and sign-in. Kratos's code sign-in still tells API
callers when an address has no account; that residual risk is accepted, slowed by per-IP and
per-address limits, and Kratos's experimental mitigation stays off.

**Why.** The mitigation doesn't cover the code sign-in path, and Shell can't close it without leaving
the provider's flow.

**Trade-offs.** An attacker can learn which addresses have no account, at the rate the limits allow.

**Considered.** Kratos's mitigation: experimental and misses this path. Generic answers for unknown
addresses: Later.

**See.** [spec §12](spec/auth.md#12-errors-and-privacy) (AUTH-ERROR-003, AUTH-ERROR-004);
[architecture §11](architecture.md#11-security-controls-rate-limiting-and-threat-model).

### DEC-014 Sign-up terms are written at deployment, not in code

**Decision.** The terms step shows items written at deployment (terms of service, privacy policy and
required confirmations such as a minimum age), the API refuses an account until it accepts the
required ones, and Shell records each version accepted. No age or country rule is in code.

**Why.** Terms and minimum ages belong to each fork's operator and its users' countries.

**Trade-offs.** A deployment can't start until its terms are written.

**Considered.** Terms and a minimum age in code: right for one fork at most.

**See.** [spec §3](spec/auth.md#3-sign-up-sign-in-and-linking) (AUTH-SIGN-010, AUTH-SIGN-011);
[README, Deployment Decisions](../README.md#deployment-decisions).

### DEC-015 Staff sign in with passkeys only and are managed only by an operator

**Decision.** Staff sign in only with passkeys, with no self-registration, email codes or self-service
recovery, and high-risk admin actions need a fresh passkey confirmation. Only an operator, through the
cloud IAM with MFA, adds, re-enrolls, re-addresses or removes staff, with the provisioning script; no
page can, and every command is audited and announced to every admin.

**Why.** Staff can act on every account, so they stay above the email ceiling, and a stolen admin
session can't create or keep admins.

**Trade-offs.** Lost passkeys need an operator. The workforce realm keeps Kratos's recovery flow on
only so admin-issued enrollment codes work; staff addresses have no recovery role.

**Considered.** Email codes or recovery for staff: puts admin rights under the email ceiling. Staff
management in the console: a stolen admin session could add admins.

**See.** [architecture §9](architecture.md#9-administration);
[spec §10](spec/auth.md#10-staff-and-administration).

## Clients, tokens and sessions

### DEC-016 Browsers go through a BFF that keeps tokens in Strict cookies

**Decision.** FastAPI is the backend-for-frontend for browsers: it runs the code flow with PKCE and
keeps the tokens in `HttpOnly`, `SameSite=Strict`, `__Host-` cookies, with CSRF tokens as a second
layer. Mobile apps are public clients signing in through the system browser with PKCE, and they and
agents send bearer tokens to their own host.

**Why.** Browser script never sees a token, and Strict cookies travel on no cross-site request; only
the sign-in transaction cookie is Lax, since the return from the provider is cross-site.

**Trade-offs.** The BFF sits on every browser request, and Strict cookies after a cross-site return are
proved in each browser engine. Native reference test clients prove the mobile contract on iOS and Android.

**Considered.** Tokens in browser storage: readable by injected script. Lax token cookies: sent on
cross-site navigations. Embedded web views in apps: the app could read the user's credentials.

**See.** [README, Clients, Scopes and Audiences](../README.md#clients-scopes-and-audiences);
[architecture §7](architecture.md#7-token-and-cookie-flow);
[spec §8](spec/auth.md#8-sessions-tokens-and-cookies).

### DEC-017 Confidential clients authenticate with signed assertions

**Decision.** Every confidential client, the web and admin BFFs included, authenticates with
`private_key_jwt`; no client holds a shared secret.

**Why.** The private key never leaves the client, and each assertion is single-use, so a leaked request
or configuration copy can't be replayed.

**Trade-offs.** Each client's key is generated, stored and rotated.

**Considered.** Client secrets: shared with the provider and replayable if leaked.

**See.** [architecture §4](architecture.md#4-clients-scopes-and-the-principal);
[spec §9](spec/auth.md#9-clients-scopes-and-route-gates) (AUTH-CLIENT-002).

### DEC-018 Refresh tokens rotate with no grace period

**Decision.** Every refresh rotates the token, and reusing a rotated one ends the whole chain at once.
The BFF serializes refreshes per session, and mobile clients keep one refresh in flight.

**Why.** Any replay of a stolen refresh token is caught, and serializing keeps parallel requests across
replicas from tripping the reuse check.

**Trade-offs.** A refresh lost in transit signs the user out; such sign-outs are counted, and a short
grace period is Later.

**Considered.** A grace window of a few seconds: tolerates races but lets a replay through.

**See.** [architecture §7](architecture.md#7-token-and-cookie-flow);
[spec §8](spec/auth.md#8-sessions-tokens-and-cookies) (AUTH-TOKEN-007).

### DEC-019 Authlib's low-level client runs the browser code flow

**Decision.** The BFF runs the browser code flow with Authlib's low-level `AsyncOAuth2Client`, a
protocol-level client, never hand-written protocol code; HTTPX calls the providers' admin APIs.

**Why.** Authlib brings no user model or routes of its own, so the domain stays framework-free; Google
runs inside the provider, so Shell needs a client only for its own provider.

**Considered.** A consolidated auth framework such as fastapi-users: imposes its own user model and
routes.

**See.** [README, Technology Direction](../README.md#technology-direction);
[architecture §5](architecture.md#5-ports-and-adapters).

### DEC-020 PyJWT verifies access tokens

**Decision.** Shell verifies access tokens locally with PyJWT, for verification only.

**Why.** Shell issues no tokens, so it needs only a small, widely used verifier.

**Considered.** joserfc: its gain, one JOSE library for signing and verifying, matters only to an
issuer.

**See.** [architecture §3](architecture.md#3-identity-realms);
[spec §8](spec/auth.md#8-sessions-tokens-and-cookies).

### DEC-021 Every route declares its scopes, and anything undeclared is denied

**Decision.** Token verification turns any token into one principal (user or service, realm, subject,
scopes, audience). Each API has one audience, scopes are `resource:action` declared in code, and every
route declares its scopes or is marked own-session, public or internal; startup refuses any other.
The spec's route catalogue lists application routes and their gates, and CI checks it against the code.

**Why.** A forgotten check stops startup rather than opening a route, and one principal keeps every kind
of caller on one authorization path.

**Trade-offs.** Every fork route needs its declarations and a catalogue row.

**See.** [README, Clients, Scopes and Audiences](../README.md#clients-scopes-and-audiences);
[architecture §4](architecture.md#4-clients-scopes-and-the-principal);
[spec §9](spec/auth.md#9-clients-scopes-and-route-gates).

### DEC-022 The API is versioned in its URL with an OpenAPI contract

**Decision.** The API lives under `/v1`, grows only additively within a published version, keeps earlier 
versions for a published deprecation window, and publishes OpenAPI as its contract, from which the web 
and admin clients are generated.

**Why.** Mobile apps can't be forced to update, so the API must not break them.

**Trade-offs.** A breaking change needs a fresh version kept beside the old one.

**See.** [README, Clients, Scopes and Audiences](../README.md#clients-scopes-and-audiences);
[architecture §4](architecture.md#4-clients-scopes-and-the-principal).

### DEC-023 Shell runs the consent and sign-out steps

**Decision.** From the first sign-in, in every environment, Shell runs the
provider's consent step: it grants first-party clients at once, stamps a
sign-in ID and creates the sign-in's device record. The reference UI serves
only sign-in, sign-up, sign-out and settings screens, and only in
development. From launch Shell also runs the sign-out step, ending the
provider session at once only for sign-outs its clients start and asking
the user to confirm any other request.

**Why.** Every user token must carry a sign-in ID tied to a device record (DEC-024), so this can't wait
for launch; confirming unknown sign-outs stops other sites logging users out (ASVS 10.6.2).

**Trade-offs.** No sign-in works in development without Shell running.

**Considered.** The reference UI's consent step during development: tokens without sign-in IDs, so
device records and gates couldn't be built against real tokens.

**See.** [architecture §7](architecture.md#7-token-and-cookie-flow);
[spec §2](spec/auth.md#2-identity-provider-capabilities) (AUTH-IDP-017),
[§7](spec/auth.md#7-account-settings-devices-and-deletion) (AUTH-ACCT-011);
[roadmap, stage 2](roadmap.md#stage-2-auth-adapters-and-the-signed-in-session).

### DEC-024 Every access token carries its sign-in's time, steps and ID

**Decision.** Hydra's token hook copies `auth_time`, `amr` and the sign-in ID from the ID-token claims
into every access token, refreshes included, and refuses a sign-in whose device record is gone or past
its limits. If the hook fails, no token is issued.

**Why.** Gates read the latest sign-in from the token and the per-request check finds the device record
by the sign-in ID, so a token without them, or from a logged-out device, must not exist.

**Trade-offs.** Issuing tokens depends on Shell's hook. If the sign-in ID doesn't survive every refresh
(To verify), logging out one device falls back to logging out all (architecture §15).

**Considered.** Issuing tokens when the hook fails: tokens that pass no gate or skip the device check.

**See.** [architecture §5](architecture.md#5-ports-and-adapters),
[§15](architecture.md#15-to-verify-before-launch);
[spec §8](spec/auth.md#8-sessions-tokens-and-cookies) (AUTH-TOKEN-006, AUTH-SESSION-006).

### DEC-025 A session is a device record, judged at every check and continued only by its device

**Decision.** Each sign-in has one device record in "Where you're signed in", judged at every check by
its sign-in and last-active times against the session limits, the absolute limit counting from the
session's first sign-in. Only the device holding a session continues it through a fresh sign-in, by
handing back a single-use continuation code; logging out a device also ends its provider session. The
provider's own sign-in session is brief, so every session starts from a fresh sign-in, and concurrent
sessions are uncapped.

**Why.** Expiry read from stored times never waits for a sweep (ASVS 7.4.1), and a copied token can't
extend a session or reset its limit. A brief provider session keeps a remembered sign-in from starting
a session nobody signed into.

**Trade-offs.** One indexed lookup per request; a device that never hands its code back keeps two
entries until their limits end them.

**Considered.** Continuing a session from the ID token: copyable. Counting the absolute limit from the
latest fresh sign-in: a session could live for ever. A cap on concurrent sessions: Later.

**See.** [architecture §7](architecture.md#7-token-and-cookie-flow),
[§8](architecture.md#8-account-management-and-sign-out);
[spec §7](spec/auth.md#7-account-settings-devices-and-deletion),
[§8](spec/auth.md#8-sessions-tokens-and-cookies).

## Account protection

### DEC-026 Each route declares a two-part gate

**Decision.** Beside its scopes, every route declares a gate in two parts, used alone, together or not
at all: a fresh sign-in within the route's window whatever the switch says (zero meaning every time),
and a trusted device, needed only while the switch is on (DEC-027) and optionally limited to some
methods. Checks read the latest sign-in step from the last `amr` entry, fail closed on a missing or
unknown one, and refuse agent clients even right after the user signs in for one. Shell's own account
changes use the fresh-sign-in part, and each fork chooses the gate for every action it adds.

**Why.** A copied session can't pass a fresh sign-in without the mailbox or a passkey. Reading only the
latest step stops an earlier step from vouching for a later one (ASVS 6.8.4), and one declaration with
two client handlers lets forks gate actions without extra screens.

**Trade-offs.** Agents can't do gated actions. With a provider that can't show the latest step, a
passkey can't stand in and no gate can be limited to passkeys.

**Considered.** An assurance level on the principal: meaningless under the email ceiling (DEC-007). A
per-route option needing a trusted device even with the switch off: Later.

**See.** [architecture §4](architecture.md#4-clients-scopes-and-the-principal);
[spec §6](spec/auth.md#6-fresh-sign-ins-and-trusted-devices),
[§9](spec/auth.md#9-clients-scopes-and-route-gates).

### DEC-027 Trusted devices are an optional switch backed by device keys Shell builds

**Decision.** Each account has a "Require trusted devices" switch, which a fork may force on for
everyone; while it is on, a route with a trusted-device part needs a fresh signature from a device key
the browser or app can use but can't read out, enrolled on first need after a fresh sign-in. Turning
the switch on needs nothing; turning it off needs an enrolled device and a fresh sign-in; removing a
device is unchecked and immediate. Trust survives sign-out and ends only by removal or disuse.

**Why.** A stolen session can't produce the device's signature, so it can neither use risky actions nor
switch the protection off. Ory has nothing like device keys, so Shell holds the public keys itself.

**Trade-offs.** Shell builds and maintains the key handling, and a browser may lose its key, after
which the device enrolls again. Shell defines no risky actions; forks with money or sensitive data
should force the switch on.

**Considered.** Requiring trusted devices for every account in Shell: no risky action of Shell's own
justifies it. Device Bound Session Credentials: Later.

**See.** [README, Authentication Philosophy](../README.md#authentication-philosophy);
[architecture §8](architecture.md#8-account-management-and-sign-out);
[spec §6](spec/auth.md#6-fresh-sign-ins-and-trusted-devices).

### DEC-028 Unseen devices are recognised by random IDs hashed per account

**Decision.** Each browser carries a random ID in a cookie, and each app install keeps one in its
keystore and reports it in a signed-in request just after each sign-in, never in a URL. Shell stores
only hashes keyed per account and sends a notice when an account, staff included, signs in from a
device it hasn't seen.

**Why.** A mailbox thief signing in elsewhere shows up as an unseen device, the owner's main warning
under the email ceiling. Keyed hashes stop one browser linking two accounts, and the IDs stay out of
URLs and logs (ASVS 14.2.1).

**Trade-offs.** A cleared cookie or reinstalled app sends a notice. The cookie is named in the privacy
policy as a security cookie (whether it needs consent is To verify per fork).

**See.** [architecture §8](architecture.md#8-account-management-and-sign-out);
[spec §11](spec/auth.md#11-notifications-and-audit-log),
[§12](spec/auth.md#12-errors-and-privacy) (AUTH-PRIVACY-004).

### DEC-029 Deletion disables the account at once and completes after a delay only support can stop

**Decision.** "Delete account" always asks the user to sign in again, answering a single-use challenge
tied to the session and the deletion, then asks "Are you sure?". The account is disabled at once, its
sessions end and a notice goes out; it is deleted after a delay unless support cancels, and support may
keep it disabled until the owner writes from the recovered mailbox (the support freeze). An independent 
audit record precedes irreversible deletion, so a backup restore can re-apply deletions before serving 
traffic.

**Why.** A copied session can't delete the account (ASVS 7.5.3), the delay and freeze let an owner
whose mailbox was taken stop a malicious deletion, and a restore never brings deleted data back.

**Trade-offs.** The disabled user can't cancel, only support can, so the notice says how to reach them.

**Considered.** Immediate deletion: no way back from a malicious request. A cooling-off the user can
cancel: helps the mailbox holder as much as the owner.

**See.** [spec §7](spec/auth.md#7-account-settings-devices-and-deletion) (AUTH-ACCT-007–009),
[§10](spec/auth.md#10-staff-and-administration) (AUTH-ADMIN-005),
[§12](spec/auth.md#12-errors-and-privacy) (AUTH-PRIVACY-005);
[architecture §9](architecture.md#9-administration).

## Workflows and integrations

### DEC-030 Work across the provider and Shell's database runs as durable tasks

**Decision.** An operation spanning the identity provider and Shell's database, such as disabling or
deleting a user, writes a `tasks` row in the use case's Unit of Work, which arq runs with retries and a
sweep for lost jobs. Shell uses no state-machine library, and lifecycle state such as `disabled_at`
stays on its entity.

**Why.** The Unit of Work can't cover the provider, so the commit records the change, and the API acts
on it at once, while the provider steps retry.

**Trade-offs.** Provider-side effects lag until the task runs, and failures are audited and alerted.
The `tasks` table is the seam where an outbox can later take over.

**Considered.** A state machine owning workflow state: no baseline workflow needs one. Provider calls
inside the request: partial failures with no retry.

**See.** [README, State Machine Driven Workflows](../README.md#state-machine-driven-workflows);
[architecture §6](architecture.md#6-data-ownership-and-database-layout),
[§9](architecture.md#9-administration).

### DEC-031 Every external system sits behind a port with contract tests

**Decision.** Each external system is reached through a port owned by the domain or application layer
and defined before its adapter; ports with more than one plausible implementation, the identity
provider's included, carry contract tests every adapter must pass. Vendor specifics live only in
adapters and configuration, the spec states provider capabilities rather than Ory features, and
PostgreSQL and Redis use only features their managed versions support.

**Why.** Any component can be swapped by configuration or an adapter that passes the contract tests,
without redesign.

**Trade-offs.** Ports with more than one plausible implementation need fakes and contract suites,
and vendor features beyond the checklist go unused.

**See.** [README, Ports and Adapters](../README.md#ports-and-adapters);
[architecture §5](architecture.md#5-ports-and-adapters).

### DEC-032 Identity-provider email goes through Shell's EmailSender

**Decision.** The provider hands every email to Shell's `EmailSender` over HTTP, which enforces
per-address and per-IP limits and the resend gap, renders localized templates and sends through the
email adapter.

**Why.** Limits, templates and the email provider stay Shell's, whichever identity provider runs, and
Kratos has no resend gap.

**Trade-offs.** Codes depend on Shell being up; the provider retries on failure.

**Considered.** The provider's own SMTP courier: no per-address limits or Shell's templates, and tied
to the provider.

**See.** [architecture §5](architecture.md#5-ports-and-adapters),
[§11](architecture.md#11-security-controls-rate-limiting-and-threat-model);
[spec §5](spec/auth.md#5-email-codes-and-rate-limits).

## Administration

### DEC-033 The admin console is its own app on its own host

**Decision.** Administration is a React-admin app in `apps/admin/` on its own host, calling only
Shell's admin routes, which need a workforce token, the admin role and CSRF protection and audit every
request, reads included.

**Why.** Console code never ships to subscribers, and the host can be restricted at the network level.

**Considered.** Admin pages in the subscriber web app: ships admin code to everyone on a shared origin.

**See.** [README, Administration](../README.md#administration);
[architecture §9](architecture.md#9-administration).

### DEC-034 The console's Data section is the one tool for table data

**Decision.** A context registers a table, and Shell generates its admin routes and console pages
behind the staff guard, exposing only listed fields and auditing every view and edit, in every
environment. It changes rows, never schemas or the providers' data, offers no export, and no raw
database tool is part of Shell.

**Why.** Every table gets one audited, scoped path, so staff never need database access and every
access is on record (PIPA).

**Trade-offs.** Each table needs a registration; bulk export is Later.

**Considered.** A raw database tool for staff: unaudited and exposes every field.

**See.** [architecture §9](architecture.md#9-administration);
[spec §10](spec/auth.md#10-staff-and-administration) (AUTH-ADMIN-010–012).

## Internationalization and operations

### DEC-035 Languages and timezones are data

**Decision.** Times are stored and sent in UTC, every user-visible string lives in a message catalogue,
and the backend sends no interface text: clients translate errors by stable code, and `EmailSender`
renders emails in the recipient's language.

**Why.** Adding a language or timezone takes catalogues and one configuration entry, never code.

**See.** [README, Internationalization](../README.md#internationalization);
[architecture §12](architecture.md#12-internationalization).

### DEC-036 Every connection is encrypted outside development

**Decision.** Outside development, public hosts use TLS with publicly trusted certificates; internal
connections use TLS under a private CA made per deployment, or a platform that encrypts service
traffic itself, declared in configuration; PostgreSQL, Redis, the relay and managed services use
verified TLS. Development keeps plain HTTP inside Compose.

**Why.** A private network alone doesn't stop a foothold inside it from reading tokens and admin calls
(ASVS 12.3).

**Trade-offs.** Each deployment issues and renews internal certificates.

**Considered.** TLS at the edge only: internal traffic readable. Mutual TLS: Later.

**See.** [architecture §13](architecture.md#13-operations-baseline);
[spec §14](spec/auth.md#14-non-functional-requirements) (NFR-014, NFR-015).

### DEC-037 Hosting choices are made per fork at deployment

**Decision.** Region, compute platform, managed services, secrets manager, edge, email provider and 
independent audit storage are chosen when a fork is first deployed, not in Shell's design, and a swap 
is configuration or an adapter passing the port's contract tests. Shell sets no default region, and a 
deployed environment refuses to start without one. Shell is proved deployment-ready once, on a reference 
deployment that swaps a component for a managed service by configuration alone.

**Why.** Law and cost follow each fork's operator and users, and a fork develops on the defaults until
it deploys.

**Trade-offs.** Each fork's deployment makes these choices and repeats the per-deployment checks the
owning files list.

**See.** [README, Deployment Decisions](../README.md#deployment-decisions);
[architecture §5](architecture.md#5-ports-and-adapters);
[roadmap, stage 8](roadmap.md#stage-8-shell-is-deployment-ready).

### DEC-038 Fork guides carry working examples run in CI

**Decision.** How-to guides for forks live in `docs/guides/`, starting with adding a feature, and each
guide's working example runs in CI.

**Why.** A guide can't drift from the code without failing CI.

**Considered.** A command that generates a feature's skeleton: Later.

**See.** [README, Documentation](../README.md#documentation);
[roadmap, stage 7](roadmap.md#stage-7-administration).
