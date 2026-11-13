# Roadmap

The order in which Shell's baseline is built and proved. The
[README](../README.md) owns principles, [architecture.md](architecture.md)
the mechanisms, and [spec/auth.md](spec/auth.md) behaviour, values and the
requirement IDs; this file only sequences them. "arch §n" is section n of
architecture.md, "spec §n" section n of spec/auth.md.

## How the stages work

- A stage starts once the preceding stage is complete.
- Each stage writes the integration-suite tests ([arch §13](architecture.md#13-operations-baseline))
  for what it builds, and a contract-test suite
  ([arch §5](architecture.md#5-ports-and-adapters)) for ports with more than
  one plausible implementation, starting in stage 1.
- Every requirement ID in the spec is listed in exactly one stage: the one
  that builds it or, for a property of the whole system, the one that
  proves it. A later stage that builds on a requirement, or runs a part of
  its check that couldn't run earlier, names it without listing it again.
- **Exit criteria**, for every stage: each requirement passes every
  Verification kind due by that stage, including deferred parts of
  earlier requirements; CI and the stage's own checks pass.
  Requirements covering a growing set are checked as members are added.
  The final slice and the completion check on `main` verify the stage
  cumulatively, before it is tagged.
- **Spikes** run as soon as their prerequisites exist, before the work
  that relies on them. Where arch §15 gives a fallback, it applies
  without relaxing the spec. Otherwise a failed spike stops the
  dependent work until the owning design file is updated.
- Shell is built and proved once, on one reference deployment (stage 8).
  The deployment decisions, and what each fork's deployment repeats, stay
  in the files that own them (end of stage 9).

| Stage | Delivers | IDs |
|---|---|---|
| 0 | Foundation and workflow | 3 |
| 1 | Ory in production mode, configured as code | 29 |
| 2 | Auth adapters and the signed-in session | 38 |
| 3 | Identity context, gates and staff | 27 |
| 4 | Our sign-in screen | 24 |
| 5 | Account settings, devices and deletion | 33 |
| 6 | Trusted devices | 10 |
| 7 | Administration | 10 |
| 8 | Shell is deployment-ready | 11 |
| 9 | Launch hardening | 9 |

## Stage 0. Foundation and workflow

**Goal.** A repository whose tooling, CI and local environment enforce the
architecture's rules from the first commit.

**Builds.**

- The repository layout in the [README](../README.md#repository-layout),
  with `main.py`, `worker.py`, `core/` and `shared/` in their four layers.
- uv, Ruff, Pyright strict, pytest and pre-commit, and the import-linter
  contracts of [arch §14](architecture.md#14-import-rules), with a
  common quality-check script used locally and in CI.
- The CI foundation of arch §13: lint, types, tests, integration
  against Compose, image build with digest pinning and scans, and
  migration checks. Each check starts in the slice that makes it
  runnable; checks needing later features arrive with those features.
- A requirement-trace check in CI, using actual test results from the
  supported runners. It checks every automated verification kind and
  named part due for completed stages and, on `stage-N/exit`, stage N.
  The same check runs for stage N on `main` before tagging, and on its
  tag. A small checked-in inventory lists all named verification parts
  and their due stages, including deferred parts, as assigned in
  approved stage plans; the spec and roadmap own the expected coverage.
  Unknown IDs, missing results and missing passing evidence fail the
  check. Configuration checks are automated unless the approved stage 
  plan identifies a check or named part as a deployment inspection. 
  The inventory records these; owner-run verification is reviewed separately.
- The development stack in Docker Compose, started by one command:
  Caddy with a local CA, PostgreSQL with the `shell` database and its
  app and migrator roles, Redis and Mailpit, bound to `127.0.0.1`,
  with a git-ignored `.env`. Synthetic seed data arrives with its tables.
- Migration tooling and the initial schema revision; the app skeleton
  with settings, startup checks, Unit of Work, error handler
  ([arch §10](architecture.md#10-error-model)), correlation IDs, JSON
  logs, basic metrics and internal health and readiness probes.
  Readiness covers the database and schema; signing keys join it
  in stage 2.

**Requirements.** AUTH-ERROR-001; NFR-011, NFR-013.

**Exit criteria.**

- Guardrail check: a deliberate breach of each of the nine import rules,
  a Pyright strict error and a Ruff error each fail their checks.
- The requirement-trace check's tests prove that missing, skipped or
  failed verification, an unknown ID and a missing required kind or
  part cannot satisfy stage completion.
- The integration job runs meaningful tests against the Compose stack;
  its first Ory tests arrive in stage 1.
- Readiness fails without the database or a compatible schema, and
  probes and metrics cannot be reached through the public edge.

**Depends on.** Nothing.

## Stage 1. Ory in production mode, configured as code

**Goal.** Both realms run in production mode in development, configured
only from files in the repository, with email codes delivered by Shell,
so every later stage builds against the real provider.

**Builds.**

- Each realm's Kratos and Hydra in Compose, in production mode, with HTTPS
  issuers and Secure cookies through Caddy's local CA, and their own
  databases and roles ([arch §6](architecture.md#6-data-ownership-and-database-layout));
  their admin APIs on an internal network that only `api`, `worker` and
  the deploy job join ([arch §1](architecture.md#1-system-and-deployment)).
- The customer realm: email codes and Google for sign-up and sign-in;
  passkeys as a second factor only, with required AAL `aal1`; no recovery
  flow; profile editing off; the provider-side windows and lifetimes the
  spec sets. No stage builds any way to give a subscriber a new address.
- The workforce realm ([arch §3](architecture.md#3-identity-realms),
  [§9](architecture.md#9-administration)): passkey sign-in only, no
  registration, and the recovery flow on solely for operator-issued
  enrollment codes, with no recovery role on staff addresses.
- Hydra per realm: ES256 signing keys, token lifetimes, refresh rotation,
  PKCE, `private_key_jwt` and client credentials.
- The client registry, `config/clients/<realm>.yaml`, each client with its
  type, applied on every deploy.
- The reference UI for each realm, in development only.
- Passkey RP IDs and allowed origins for both realms.
- `EmailSender` behind the provider's HTTP courier, with English and Korean
  code templates, its email adapter and limits, sending to Mailpit. Its
  contract suite runs on the fake and real adapter. Notification delivery
  arrives in stage 3.
- The edge: each realm's host routed to its public paths only, per-IP
  limits on flow submissions, and the client IP it sets
  ([arch §11](architecture.md#11-security-controls-rate-limiting-and-threat-model));
  the provider's enumeration mitigation left off; baseline security headers on each 
  host, including the reference UI's CSP and `no-store` on sign-in and token responses 
  (arch §11).
- Startup checks refusing realms that share an issuer, key URL or audience.

**Requirements.** AUTH-ASSURE-001; AUTH-IDP-001–002, AUTH-IDP-005,
AUTH-IDP-011–014, AUTH-IDP-020; AUTH-CODE-001–006; AUTH-LIMIT-001–005;
AUTH-PASSKEY-003, AUTH-PASSKEY-009; AUTH-SESSION-004; AUTH-CLIENT-001–003;
AUTH-STAFF-008; AUTH-ADMIN-008; AUTH-ERROR-004.

**Exit criteria.**

- Configuration-as-code check: from empty volumes, one command brings up
  both realms with their clients and keys, and each serves discovery with
  its exact issuer, with no manual step.
- Provider-only integration tests use test fixtures; complete application
  sign-ins wait for Shell's consent step in stage 2.
- Spike: user verification. Record whether each realm's provider refuses
  a passkey response without the user-verified flag in its browser and API
  flows. If not, build and prove Shell's unavoidable server-side check in
  stage 2 before dependent passkey tests (arch §5). Use test-only provider
  flows for the spike. If the check cannot be enforced, stop until the
  design is corrected.
- Spike: the provider's own account linking at sign-in can be turned off
  (AUTH-SIGN-009, arch §15).

**Depends on.** Stage 0.

## Stage 2. Auth adapters and the signed-in session

**Goal.** Subscribers and test-fixture staff sign in and call the API with
tokens Shell verifies locally, using the reference UI where supported.
Staff fixtures are test-only, never seed data.

**Builds.**

- The remaining ports of arch §5 per realm: `TokenVerifier`, `OidcClient`,
  `IdentityAdmin` and `GeoLocator`, with fakes and contract suites on fakes
  per commit and on real adapters in integration CI.
- If the stage 1 user-verification spike requires it, Shell's server-side
  passkey check, before staff sign-in is tested.
- The `users`, `staff` and `devices` tables, keyed by Shell's UUID, never
  holding an address. They are built here because the consent step and the
  per-request check below need them.
- Token acceptance: `verify_token`, the key cache, the per-request lookup
  of the account and its device record, the principal, and one router per
  realm. Idle and absolute session limits on requests and token issuance;
  readiness covers the signing keys. Each Shell route declares its scopes
  and gate; startup refuses undeclared routes. Stage 3 adds the fresh
  sign-in gate and catalogue check.
- The audit log, its port and append-only roles, including the purge role;
  consent audits sign-ins, report hooks audit sign-ups, and sign-out
  writes its own entry.
- The `tasks` table and arq worker's unfinished-device cleanup, using the
  stored ten-minute deadline so cleanup can resume after interruption.
- Our consent step: it grants first-party clients at once, stamps the
  sign-in ID and creates the device record with its city. It runs from the
  first sign-in, because every user token must carry the sign-in ID; the
  reference UI keeps the sign-in screens until stage 4.
- The token-claims hook (`auth_time`, `amr` and the sign-in ID in every
  access token, refreshes included, failing closed), built before any
  account change because every gate reads its claims.
- The BFF on both browser hosts: sign-in, callback, serialized refresh and
  sign-out, with the cookies and CSRF protection of
  [arch §7](architecture.md#7-token-and-cookie-flow); bearer tokens only on
  `api.`.
- The customer profile read route, returning the caller's name, language
  and timezone under `profile:read`.
- The first substantive OpenAPI contract, with CI validation and
  compatibility checks once a published baseline exists.
- The React web shell, with its frontend tooling, English and Korean
  FormatJS catalogues, missing-key check, refresh handling, CSRF protection
  and sign-out. CSP for `app.` and `no-store` on sign-in, token and account responses.
- The provider's hook routes on the private network, receiving sign-up
  and sign-in reports.
- The DB-IP City Lite file and its monthly refresh, in place before the
  device list is built.
- Behaviour with the provider down (arch §10).
- Two CI checks that every later stage passes: a scan of the integration
  suite's logs for personal data, and a schema check for personal fields.

**Requirements.** AUTH-IDP-009, AUTH-IDP-015–018; AUTH-LIMIT-006; AUTH-ACCT-010;
AUTH-DEVICE-002–003; AUTH-SESSION-001–003, AUTH-SESSION-005–006;
AUTH-TOKEN-001–007, AUTH-TOKEN-009–010; AUTH-WEB-001–004, AUTH-WEB-006–007;
AUTH-CLIENT-007–008; AUTH-AUDIT-001–004; AUTH-PRIVACY-002; NFR-002, NFR-010.

**Exit criteria.**

- Spike, first: the sign-in ID our consent step stamps survives every
  refresh ([arch §15](architecture.md#15-to-verify-before-launch)). Pass:
  across a chain of ten refreshes, every access token carries the sign-in
  ID stamped at sign-in. If it fails, arch §15 says what follows.
- Runs the parts that couldn't run earlier: AUTH-IDP-012 (code reuse),
  NFR-013 (keys). Verify the workforce sign-in path with test fixtures.
  Run complete application token flows here, not in stage 1.

**Depends on.** Stage 1.

## Stage 3. Identity context, gates and staff

**Goal.** Every route declares who may call it and how recent a sign-in it
needs, every security-relevant action is audited, and an operator can
manage staff, all before the first account change.

**Builds.**

- The `admin` role, referencing staff only, and the admin guard on
  `/v1/admin`, with its first route: reading one's own staff record.
- Shell's full scope set, with the own-session sign-out route for apps
  and agents.
- Client types read from the client registry.
- Gates ([arch §4](architecture.md#4-clients-scopes-and-the-principal)):
  `require_gate` on every route, the fresh-sign-in check reading the latest
  step from the last `amr` entry, the refusal of agents, the RFC 9470
  challenge and the gate error codes. The trusted-device part is declared
  here and enforced in stage 6; until then it passes, as with the switch
  off.
- A test-only route with a 15-minute fresh-sign-in gate, to prove the
  gate before the account-management routes arrive.
- Signing in again through the reference UI, the web client's handler for
  the sign-in-again challenge, and continuation codes, so only the device
  holding a session can continue it.
- Native reference mobile test clients in `tests/mobile/`, in Swift
  and Kotlin, exercising system-browser sign-in, token storage and
  rotation, sign-out and fresh-sign-in continuation. Their integration
  suites run in CI on the iOS Simulator (macOS) and Android Emulator
  against the real stack through isolated HTTPS test hosts (arch §13).
- The CI check of the route catalogue
  ([spec §9](spec/auth.md#9-clients-scopes-and-route-gates)) against the
  code, which every later stage's routes pass.
- Audit retention purge.
- Durable work: extend the task mechanism with the remaining sweeps and
  coordinated disable and enable (arch §9), ahead of the console's
  disable and delete actions.
- Notices queued as tasks, with their templates. The staff script is the
  first change that sends one.
- The staff provisioning script (`add`, `re-enroll`, `change-email`,
  `remove`), with staff enrolling and signing in through the workforce
  realm's reference UI. The workforce realm and the script come before the
  console.

**Requirements.** AUTH-ASSURE-002; AUTH-IDP-004, AUTH-IDP-007; AUTH-FRESH-001–002,
AUTH-FRESH-004–005, AUTH-FRESH-009–010; AUTH-TOKEN-008; AUTH-CLIENT-004–006;
AUTH-SCOPE-001, AUTH-SCOPE-003–004; AUTH-STAFF-001–007; AUTH-ADMIN-001;
AUTH-NOTIFY-005–006; AUTH-ERROR-002.

**Exit criteria.**

- Spike, first (arch §15): Google can be made to ask again. Pass: signing
  in again through Google makes Google ask the user, and the tokens carry a
  fresh `auth_time`. If it fails, arch §15 says what follows.
- Spike, first (arch §15): a passkey confirmation through Hydra works
  on its own. With a test-fixture subscriber, prove a fresh `auth_time`
  and `amr` ending in `webauthn`, then in `code` after signing in again
  by email code. Prove AUTH-IDP-007 before building gates.
- Prove the latest authentication method before building gates; prove staff
  passkey sign-in and enrollment with software authenticators before using
  the provisioning script. Real-device checks run in stage 6.
- Runs the parts that couldn't run earlier: AUTH-SESSION-001–003 (fresh
  sign-in without extending the absolute limit); AUTH-AUDIT-004 (purge);
  AUTH-ACCT-010 (own-session sign-out); AUTH-TOKEN-007 and AUTH-TOKEN-009 
  (mobile refresh and token storage).

**Depends on.** Stage 2.

## Stage 4. Our sign-in screen

**Goal.** Our own screens take over every sign-in step from the reference
UI, so deployed environments can serve them.

**Builds.**

- The sign-in module in `web/`, on each realm's auth host: the customer
  entry screen (the email step and "Continue with Google"), code entry,
  sign-up; signing in again; staff sign-in with
  passkeys and staff enrollment; the provider's settings steps, used from
  stage 5; the message for a disabled account.
- Google sign-up rules: our claims mapper and sign-up hook, each refusal
  with its message, and each method finding the account by its own key.
- CSP for each auth host. The terms page on the app host, after the BFF
  callback: items configured at deployment, placeholders only in
  development, the API refusing the account until the required ones are
  accepted, and a record of each item. Mobile clients accept terms through 
  the API. All terms responses carry `no-store`.
- Our app takes over the sign-out step: our clients' sign-outs are accepted
  at once, any other request is confirmed, and Hydra then ends its sign-in
  session and the Kratos session.
- Every passkey ceremony requires user verification and accepts security
  keys; screens answer alike for known and unknown addresses.
- Extend the English and Korean FormatJS catalogues to the sign-in screens
  and error codes, with CI failing on a missing key. Screens are built to WCAG 2.2 AA, with
  the automated checks of AUTH-A11Y-001 on each.
- The reference UI kept to development.

**Requirements.** AUTH-IDP-006, AUTH-IDP-008, AUTH-IDP-019; AUTH-SIGN-001–013;
AUTH-PASSKEY-001–002; AUTH-ACCT-011; AUTH-ERROR-003; AUTH-PRIVACY-001;
AUTH-I18N-001–002, AUTH-I18N-004.

**Exit criteria.**

- Hands-on check (arch §15): the provider doesn't reveal a disabled account
  before its code is entered. Pass: a disabled account's address gets the
  same screen and response as any other until the code is entered. If it
  fails, arch §15 says what follows.
- Runs the parts that couldn't run earlier: AUTH-FRESH-004 (screen);
  AUTH-ACCT-010 (our sign-out step); AUTH-IDP-004 (our screen).

**Depends on.** Stage 3: the terms step relies on route declarations,
signing in again on the sign-in checks, and the disabled-account message on
coordinated disable.

## Stage 5. Account settings, devices and deletion

**Goal.** A subscriber manages their account, sees and ends their sessions
and deletes the account, each change behind the gate the spec sets and
announced as the spec lists.

**Builds.**

- The settings page ([arch §8](architecture.md#8-account-management-and-sign-out)),
  showing the address without any way to edit it.
- Name, language and timezone, and the optional terms items.
- Passkeys for both realms on the provider's settings screens, served by
  our screen: adding, removing and several per account, with Shell's labels
  in `passkey_labels` for renaming. Our settings hook applies the windows and refuses a change
  whose latest step is a passkey.
- Linking and unlinking Google, in settings only. Provider settings-change
  report hooks; `no-store` on account responses.
- "Where you're signed in": the list, with city and the DB-IP credit;
  logging out another device or all devices.
- New-device notices for subscribers and staff: the browser ID hashed per
  account in `known_browsers`, and the install ID that apps report,
  exercised by the reference mobile test client.
- Deletion: the challenge, the every-time sign-in, "Are you sure?", the
  pending deletion in `pending_changes` and the task that records the
  irreversible deletion in independent audit storage (arch §6) before 
  deleting the account. Its port and durable local adapter are built here.
- The notices for passkeys, Google and deletion; sign-out on every page,
  and "Log out all devices" after a change to how the account signs in.

**Requirements.** AUTH-IDP-003, AUTH-IDP-010; AUTH-PASSKEY-004–008;
AUTH-FRESH-003, AUTH-FRESH-007–008; AUTH-ACCT-001–009, AUTH-ACCT-012;
AUTH-DEVICE-001, AUTH-DEVICE-004–006; AUTH-SCOPE-002; AUTH-NOTIFY-001–004;
AUTH-PRIVACY-003–004; AUTH-I18N-003, AUTH-I18N-005.

**Exit criteria.**

- Spike, first (arch §15): settings hooks can tell adding a passkey or
  Google from removing one. Pass: for each of the four changes, the hook's
  payload names the kind of change and the passkey or Google account it
  concerns.
- Runs the parts that couldn't run earlier: AUTH-IDP-008 for settings
  changes; AUTH-IDP-009 for settings-change reports; AUTH-IDP-004 for
  customer registration; AUTH-CLIENT-004 for the install ID; AUTH-DEVICE-003
  for the list; AUTH-ACCT-010's sign-out without a new-device notice;
  AUTH-TOKEN-008 for logging out one or all devices and deletion;
  AUTH-PASSKEY-001–002 for registrations; AUTH-NOTIFY-005 for this stage's
  templates; AUTH-SCOPE-003–004 for this stage's routes; AUTH-FRESH-004 on 
  the account-management routes; AUTH-FRESH-010 for the visible device list; 
  AUTH-CLIENT-004 for the settings screens.

**Depends on.** Stage 4.

## Stage 6. Trusted devices

**Goal.** The Require trusted devices switch works end to end, so a fork
can add risky actions without new screens.

**Builds.**

- Device keys: a non-extractable Web Crypto key in browsers, the platform
  keystore in the reference mobile test client, one per account per browser or
  install, with Shell holding the public key in `trusted_devices`.
- "Trust this device" at any route with a trusted-device part, the
  single-use challenges and the proof on each request.
- The trusted-device part of the gate, enforced; the switch per account,
  and the fork setting that forces it on.
- The trusted-device list, removal and expiry; trust kept across sign-out
  and "Log out all devices".
- The web client's handler for `auth.trusted_device_required`.
- A passkey standing in for signing in again on a trusted device with the
  switch on: the client checks the device and the switch with Shell, and
  our screen offers the passkey first, with an email code beside it.
- The test-only route with a trusted-device part, in test builds only, so
  the switch path is tested before any fork adds a risky action.
- The notices for trusted devices and the switch.

**Requirements.** AUTH-ASSURE-003; AUTH-IDP-021; AUTH-FRESH-006;
AUTH-TRUST-001–007.

**Exit criteria.**

- Spike, first (arch §15): the client's flag reaches our sign-in screen
  and selects the passkey option (AUTH-IDP-021).
- Spike, first (arch §15): device keys. Pass: a non-extractable Web Crypto
  key survives restarts and updates in Safari, Chrome and Firefox, and the
  iOS and Android keystores keep their key across both; what Safari's
  deletion of script-writable storage and private windows do to keys is
  recorded in arch §8, with the device enrolling again when its key is gone
  (AUTH-ASSURE-003).
- Hands-on check (arch §15): devices and browsers, run here against the
  isolated HTTPS test stack of arch §13, because all its subjects now
  exist. Pass: on iOS, Android, macOS and Windows, staff passkey sign-in 
  works with autofill, and passkey confirmations and staff sign-in work 
  with a phone through the cross-device flow and with a security key, 
  each asking for user verification; code entry works on phones; 
  the reference mobile test client signs in through the system browser; 
  Google sign-in works in Safari, Chrome and Firefox. This runs the 
  hands-on parts of AUTH-IDP-004, AUTH-PASSKEY-001–002 and AUTH-SIGN-012.
- Runs the parts that couldn't run earlier: AUTH-ERROR-002's
  trust-this-device handler; AUTH-FRESH-003's windows for enrolling and for
  turning the switch off; AUTH-ACCT-001's trusted-device and switch items;
  AUTH-NOTIFY-001 for trusted devices and the switch; AUTH-PRIVACY-003 for
  trusted devices; AUTH-PASSKEY-001–002 for confirmations;
  AUTH-SCOPE-003–004 for this stage's routes; AUTH-ACCT-010 for device-key
  retention; AUTH-CLIENT-004 for mobile device keys and passkey confirmations;
  AUTH-LIMIT-006 for passkey confirmations; AUTH-SESSION-003 for the absolute 
  limit across confirmations; AUTH-DEVICE-005 for retaining trusted devices; 
  AUTH-ACCT-009 for deleting trusted-device records.

**Depends on.** Stage 5 and the device-key spike.

## Stage 7. Administration

**Goal.** Staff support subscribers and manage registered tables from the
console, with the passkey confirmations, confirmation steps and audit the
spec requires.

**Builds.**

- The React-admin console in `apps/admin/` on the admin host, with its own
  translation provider, sign-out on every page and the strictest CSP.
- The admin endpoints of arch §9: subscribers listed, searched and viewed
  with their audit entries and devices; created and edited; disabled,
  enabled and deleted through the coordinated disable of stage 3 and the
  deletion of stage 5; a pending deletion cancelled, or kept disabled under
  the support freeze; all of a subscriber's sessions and grants ended.
- A passkey confirmation before each high-risk action, a `confirm` field on
  destructive ones, and an audit entry for every request, reads included.
- Staff's own "Where you're signed in", with logging out, and the reminder
  to add a second passkey.
- The notices for support actions.
- The Data section (arch §9), with Shell's own tables registered as the
  spec lists.
- `docs/guides/adding-a-feature.md`, whose working example is the test
  context of AUTH-ADMIN-010, run by CI on every change.

**Requirements.** AUTH-STAFF-012; AUTH-ADMIN-002–007, AUTH-ADMIN-010–012.

**Exit criteria.**

- Guide check. Pass: following only the guide, someone who didn't build the
  Data section adds a small feature with a table, and it appears in the
  console with its audited pages.
- Runs the parts that couldn't run earlier: AUTH-ACCT-004 for admin routes;
  AUTH-ACCT-012 and AUTH-ADMIN-008 for the console; AUTH-FRESH-003 for staff
  high-risk actions; AUTH-NOTIFY-001 for support actions;
  AUTH-SCOPE-003–004 for the workforce routes; AUTH-TOKEN-008 for admin
  revocations.

**Depends on.** Stage 6.

## Stage 8. Shell is deployment-ready

**Goal.** Prove that Shell deploys to a non-local environment in an
explicit region, with a component swapped for a managed service by
configuration alone and every connection encrypted.

**Builds.**

- A reference deployment on the default compute platform
  ([README](../README.md#deployment-decisions)), starting empty, with our
  sign-in module, the email relay and a secrets manager; its first admin is
  added with `add`.
- Internal TLS with this first deployment
  ([arch §13](architecture.md#13-operations-baseline)): a private CA made
  for the deployment, each service's certificate issued and renewed by the
  deploy job, and the startup checks that refuse plaintext.
- Public TLS on every host; verify security headers on all deployed hosts.
- Backups of every database, the providers' included.
- The deployed independent audit store, passing the deletion-record
  contract tests from stage 5, and the restore job that re-applies
  deletions before serving traffic (arch §13).
- Logs shipped off the host, metrics with their alerts, the sending
  domain's mail records, and the app's least-privilege workload identity.

**Requirements.** AUTH-WEB-005; AUTH-STAFF-011; AUTH-ADMIN-009; AUTH-NOTIFY-007;
AUTH-PRIVACY-006; NFR-006, NFR-008–009, NFR-012, NFR-014–015.

**Exit criteria.**

- **Deployment-ready test.** Pass: the deployment starts only once its
  region is set; PostgreSQL is swapped for a managed service in the same
  region by configuration alone, with no change to domain or application
  code, and sign-in, settings and the console then work on it; each startup
  refusal in arch §5's table is exercised, the deployment refusing to start
  with each faulty setting; and with the private CA, no internal endpoint
  accepts plaintext.
- Runs on the deployment the parts that couldn't run earlier:
  AUTH-SIGN-001 and AUTH-SIGN-011 (no reference UI, no placeholder terms);
  from outside the private network, NFR-013; and from outside it and from
  the edge, metrics and datastores, AUTH-ADMIN-008.

**Depends on.** Stage 7.

## Stage 9. Launch hardening

**Goal.** Prove on the reference deployment that the baseline is secure,
recoverable and fast enough before any fork launches from it.

**Builds.**

- At least two admins, each with two passkeys.
- The rehearsals and reviews below.

**Requirements.** AUTH-STAFF-009–010; AUTH-PRIVACY-005; AUTH-A11Y-001;
NFR-001, NFR-003–005, NFR-007.

**Exit criteria.**

- Load test (arch §15) on the reference deployment. Pass: the targets of
  NFR-003 to NFR-005 are met, and stopping one `api` replica fails only its
  in-flight requests (NFR-001).
- Restore rehearsal. Pass: one full restore of every database re-applies
  later deletions from the independent audit records before serving
  traffic, refuses startup if replay fails, and lets an unaffected
  restored account sign in. The duration is recorded.
- Signing-key rotation rehearsal, by the procedure in arch §13. Pass:
  tokens signed by either key verify throughout (AUTH-TOKEN-010), and no
  key set is ever emptied.
- Break-glass rehearsal (AUTH-STAFF-010).
- WCAG 2.2 AA review, in English and Korean, of every screen AUTH-A11Y-001
  lists.
- Threat-model review. Pass: every threat in arch §11 meets a mitigation
  that is built and tested, or a residual risk the file accepts; a threat
  found in the review is added to arch §11 before launch.
- ASVS 5.0 Level 2 checklist. Pass: every Level 2 requirement is met by a
  passing spec requirement, is the relaxation documented in
  [spec §1](spec/auth.md#1-conventions-and-assurance), or is recorded as not
  applicable with its reason.
- External review: an independent security review of the code and the
  reference deployment ends with no open critical or high finding.

**Depends on.** Stage 8.

**What each deployment repeats.** A fork's own deployment makes the choices
in the README's [Deployment Decisions](../README.md#deployment-decisions)
and repeats what the owning files set out for every deployment: its private
CA, backups, key rotation and DB-IP refresh
([arch §13](architecture.md#13-operations-baseline)), and the admins,
backups and availability figures of the
[spec](spec/auth.md#14-non-functional-requirements). This roadmap doesn't
repeat them.

## Open questions

None.

## Later

Nothing Later is a stage; an item is planned when its trigger appears. The
items and their triggers are in the README's
[Planned Evolution](../README.md#planned-evolution),
[arch §16](architecture.md#16-later) and [spec §15](spec/auth.md#15-later).
