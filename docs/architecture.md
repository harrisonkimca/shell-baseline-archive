# Architecture

How Shell is built. The [README](../README.md) owns principles,
[spec/auth.md](spec/auth.md) behaviour and values, [roadmap.md](roadmap.md)
the sequence, [decisions.md](decisions.md) durable decisions. Hostnames are
examples; section 15 lists every **To verify**.

## 1. System and deployment

```
PUBLIC    Subscribers: web · mobile · agents      Staff: admin console
 ┌──────────────────────────────────────────────────────────────────┐
 │ HTTPS edge: TLS · headers · per-IP limits · public paths only    │
 └────────┬───────────────────────┬────────────────────────┬────────┘
INTERNAL  │ auth.                 │ app. · api. · admin.   │ staff-auth.
 ┌────────▼────────┐   ┌──────────▼───────────┐   ┌────────▼────────┐
 │ Customer realm  │◄─►│ Shell (one image)    │◄─►│ Workforce realm │
 │ Kratos · Hydra  │   │ api ×N · worker      │   │ Kratos · Hydra  │
 │ sign-in screens │   │ BFF · API · hooks    │   │ sign-in screens │
 └────────┬────────┘   └──────────┬───────────┘   └────────┬────────┘
          ▼                       ▼                        ▼
  Kratos, Hydra DBs       shell DB · Redis         Kratos, Hydra DBs
                             email relay

CLOUD IAM  operators with MFA · CI deploy role · secrets manager · app identity
◄─► Shell calls each realm's admin API; each realm calls Shell's hooks.
```

- **Public:** only the edge listens, routing `app.`, `api.` and `admin.` to
  Shell and each auth host to its realm, whose screens and endpoints then
  share an origin and need no CORS (Kratos's and Hydra's public paths don't
  overlap). Where Shell enforces passkey user verification, the edge routes
  those submissions through Shell instead (section 5). Only explicitly
  allowed public paths are routed; admin APIs, hooks, probes and metrics
  never are.
- **Internal:** Shell, the providers, PostgreSQL, Redis and metrics on a
  private network, the only place Shell's hook routes (per-call secret,
  constant-time check) exist. Provider admin APIs sit on a separate
  internal network that only `api`, `worker` and the deploy job join, so
  the edge, the one component facing the internet, can't reach them.
  Outside development every connection there is encrypted (section 13).
- **Cloud IAM:** consoles, deploys and secrets, for operators with MFA,
  CI's federated role and the app's least-privilege workload identity; no
  realm reaches it.

Development runs everything in Docker Compose on `localhost`; deployed
environments run the same image with components self-hosted or managed
([README](../README.md#deployment-decisions)). Section 13 has details.

## 2. Layers and bounded contexts

```
apps/api/src/shell_api/
  main.py, worker.py   entry points
  core/                settings, composition root
  shared/              domain/ application/ infrastructure/ presentation/
  contexts/identity/   the same four layers
```

- **identity**, the only baseline context, owns users, staff, roles and
  the provider adapters, the only code that calls a provider: `idp/oidc/`
  for the standard protocols and `idp/ory/` for the default provider, both
  in its infrastructure layer. Its presentation layer has `bff/`
  (`/v1/auth`), `account/` (`/v1/account`), `ory/` (hooks, consent and
  sign-out steps, the only code parsing Ory payloads) and the **admin
  endpoints**, `admin/` (`/v1/admin`, admin host only), as staff and roles
  are identity. Forks add `/v1/admin/<context>` behind the same guard.
- **shared** holds the principal, errors, Unit of Work, `EmailSender`, rate
  limits, durable tasks, route dependencies and the **audit log** (port in
  `shared.application`), so any context audits inside its own Unit of
  Work. **core** wires adapters to ports; section 14 enforces this.

## 3. Identity realms

| | Customer realm | Workforce realm |
|---|---|---|
| Members | Subscribers (`users`) | Staff (`staff`) |
| Sign-in | Email codes, Google | Passkeys only |
| Sign-up | Self-service; email code or Google | None (section 9) |
| Recovery | None: an email-code sign-in is the way back | Operator re-enroll (section 9) |
| Passkeys | Second factor, confirmation only (Kratos `webauthn`, `passwordless: false`) | First factor, sign-in (Kratos `passkey`) |
| Issuer | `https://auth.<domain>/` | `https://staff-auth.<domain>/` |
| Audience | `https://api.<domain>` | `https://admin.<domain>` |
| Hosts | `app.`, `api.` | `admin.` |
| Cookies | `__Host-shell-*` | `__Host-shell-admin-*` |
| WebAuthn RP ID | The registrable domain | `staff-auth.<domain>` |

Each realm is a Kratos and Hydra pair with its own databases, or a managed
tenant, set by configuration; startup refuses realms sharing an issuer, key
URL or audience. Subjects are public, not pairwise (Hydra's default).

**Accepting a token.** Each router trusts one realm:
`verify_token(token, realm)` requires ES256, a known `kid`, a valid
signature, the realm's issuer and audience, `exp`, `nbf` and `iat` within
60 s, and `sub`, `jti`, `client_id`, scopes and, for users, `auth_time`,
`amr` and the sign-in ID (section 8). The resolver then maps (`iss`,
`sub`) to a row and the sign-in ID to its device record, refusing unknown
staff, disabled users and logged-out devices, so another realm's token
fails before any role check:

| Path to admin | Blocked by |
|---|---|
| Customer token, cookie or key | Workforce keys and cookie names only |
| Asking for the admin audience | No customer client may |
| Role via a profile edit | Roles reference `staff` only |
| Staff sign-up, code or recovery | Disabled in the workforce realm |
| Calling provider admin APIs | Internal network only |
| Stored script in staff views | Escaping, strict CSP |

## 4. Clients, scopes and the principal

Clients are configuration (`config/clients/<realm>.yaml`: type, grants,
redirect URIs, scopes, audience, first-party flag) applied on every
deploy; confidential ones sign assertions (`private_key_jwt`) instead of
holding secrets.

| Client | Type and grant | Tokens kept |
|---|---|---|
| Web and admin BFFs | Confidential; code + PKCE | `HttpOnly` cookies |
| Mobile | Public; code + PKCE | Refresh in keychain |
| Agent as itself | Client credentials | Access token only |
| Agent for a user | As mobile | As mobile |

Browser hosts accept only cookies, and unsafe methods need `X-CSRF-Token`
matching the CSRF cookie plus a same-origin `Sec-Fetch-Site` or `Origin`;
`api.` accepts only bearer tokens. Scopes are `resource:action`, declared 
in code (Shell's in the spec); a route without `require_scopes(...)`, 
`own_session` (any valid user token, acting only on its own session), 
`public` or `internal` stops startup. Internal routes have their own 
access controls (section 1). Services reach only routes marked 
`allow_service=True`.

**The principal**, a frozen value in `shared.domain`, is all the app sees:
`kind`, `realm`, `id` (our UUID or a client ID), `client_id`, `audience`,
`scopes`, `token_id`, `sign_in_id` (section 8), `auth_time`, the time of the
last sign-in or confirmation, and `methods` (`amr`): `code`, `oidc` and
`webauthn` in the customer realm, and `passkey` in the workforce realm,
where an enrollment-code session carries the recovery method
(`code_recovery`), so the admin guard refuses any workforce session not
signed in with a passkey.

**Gates.** Beside its scopes, every route, Shell's and every fork's,
declares a gate with `require_gate(fresh_sign_in=..., trusted_device=...)`,
using either part, both, or `ungated` for neither; startup refuses a route
without one, as for scopes. The spec sets each window and default.

- **Fresh sign-in**, required whatever the Require trusted devices switch
  says: an `auth_time` within the route's window, or a challenge to sign in
  again (section 10). A window of zero means every time, as for deletion:
  the sign-in must follow a single-use, short-lived challenge tied to that
  session and route, or the route refuses. Shell's own account changes use
  this part (section 8), and so do staff high-risk actions, which in the
  workforce realm only a passkey meets.
- **Trusted device**, required only while the account's switch is on: a
  fresh proof from an enrolled device key (section 8) and a sign-in within
  the route's window, optionally limited to some methods, such as passkey
  only. In the customer realm a method limit belongs only in this part,
  since passkeys confirm only on trusted devices; elsewhere it would lock
  out users without one.

Both parts refuse the agent client types in the client registry, looked up
by `client_id`, even right after the user signs in for one; with the switch
off, scopes alone decide what an agent may do. Both read the latest sign-in
step from the last entry of `amr`, never from a method found anywhere in it:
a missing or unknown latest step passes neither part, so the user is asked
again, and a `webauthn` step passes only with a proof from an enrolled
device while the switch is on. With a provider that can't show the latest
step, a passkey can't stand in for signing in again, and no gate can be
limited to passkeys. The client has one handler for each challenge (sign in
again; trust this device), so a fork's new gated route needs no new screens.
Shell has no risky action of its own, so a test-only route carries the
trusted-device part.

**Versioning.** Once published, `/v1` only grows; anything else needs
`/v2`. Superseded versions stay at least **6 months** after their
successor, with `Deprecation` and `Sunset` headers once announced.
OpenAPI is the committed contract: CI validates it and blocks
unannounced breaking changes to published versions. Web and admin
use a generated client.

## 5. Ports and adapters

- `TokenVerifier` (PyJWT): `verify_token(token, realm)`, `keys_ready(realm)`.
- `OidcClient` (Authlib): `start_sign_in`, `finish_sign_in`, `refresh`,
  `revoke`, `end_session_url`.
- `IdentityAdmin` (Kratos and Hydra admin APIs): `create_user`, `get_users`,
  `find_user_by_email`, `change_email` (the staff script's `change-email`),
  `disable_user`, `enable_user`, `delete_user`, `list_sessions`,
  `revoke_sessions` (sign-in sessions), `revoke_session` (one sign-in's
  provider session, by the login session ID our consent step recorded),
  `revoke_refresh_grants` (login and consent sessions, ending every refresh
  token), `issue_enrollment_code`, `accept_consent`, `accept_logout`.
- `EmailSender` (templates, per-address and per-IP limits, SMTP):
  `send_code(to, purpose, code, locale)`,
  `send_notification(to, event, locale, data)`.
- `GeoLocator`: `locate(ip)`, the city of a sign-in. Its adapter reads the
  DB-IP City Lite MMDB file, whose city names include Korean, with
  `maxminddb`.

Provider ports exist per realm, each with a fake. The provider calls
`presentation/ory/` to send email, approve or refuse settings changes and
sign-ups, and report sign-ups, sign-ins and changes. Hydra's token hook also
calls it at every issuance and refresh, to copy `auth_time`, `amr` and the
sign-in ID from the ID-token claims into the access token (top level,
through `allowed_top_level_claims`), and to issue nothing for a sign-in
whose device record is gone or past the session's limits (To verify: the ID
survives refreshes). Its answer replaces the token's extra claims, and if it
fails Hydra issues no token. Consent audits sign-ins and fresh sign-ins; 
report hooks audit sign-ups and settings changes, without duplicating entries.

**Contract tests**, run on fakes per commit and on real adapters in
integration CI, gate every adapter of a port with more than one plausible
implementation: token rejections, `state`, `nonce`,
PKCE, idempotent admin calls, refresh failing after both revocations
(disabling alone doesn't end it), email limits and templates.

**Capability checklist** (requirements in the spec; a candidate lacking
one is out, and cost decides the rest). Ory v26.2.0 provides:

- **Sign-in and tokens:** second-factor passkeys that can re-authenticate on
  their own (Kratos `webauthn` with `passwordless: false`), and passkey
  sign-in for staff (Kratos `passkey`); email codes for sign-up and sign-in;
  Google; the latest sign-in step as the last `amr` entry (section 4);
  required AAL settable to `aal1` for sessions and settings
  (`session.whoami.required_aal`,
  `selfservice.flows.settings.required_aal`); ES256 JWT access tokens with
  audience, scopes and JWKS, plus custom claims through the token hook; 0 s
  refresh rotation with family revocation; PKCE; `private_key_jwt` with
  ES256; client credentials; key rollover.
- **Administration and hooks:** admin disable and subject-wide revocation,
  and ending one sign-in session; admin enrollment codes; a consent step
  that can add token claims and sees the sign-in session's ID, and a
  consent-skip flag; the client's authorization request, readable by our
  sign-in screen for the passkey flag (section 8); no automatic linking;
  email over an HTTP hook, with the client IP and language; settings and
  sign-up hooks that refuse before saving, the settings ones receiving the
  session; a settable recent-sign-in window for settings; self-hosting
  anywhere.
- **Done by Shell instead:** requiring user verification in both passkey
  methods: our screen asks for it, and the server refuses a response
  without the user-verified flag. Where provider configuration cannot
  enforce this, Shell checks submissions before forwarding them to
  the provider, with no public path around the check; the provider
  still verifies the ceremony (section 15). Also accepting security
  keys in the `passkey` method, which allows device passkeys only;
  device keys and trusted-device checks.

**Deployment decisions.** Each README row plugs in through the setting
below, checked when `SHELL_ENV` is not `development`. Ory Network is
configuration; a non-Ory provider needs new adapters, a claim map and
a sign-in module. Other swaps are configuration or adapters.

| Decision | Setting | Startup refuses |
|---|---|---|
| Region | `SHELL_REGION` | No region |
| Sign-up terms | Terms-step items | Placeholder terms |
| Compute platform | Platform manifests; `SHELL_INTERNAL_TLS` | A plain-HTTP internal URL, unless the platform encrypts internal traffic |
| Identity provider | `SHELL_REALMS__*` | Shared issuers |
| PostgreSQL | `SHELL_DATABASE_URL` | No verified TLS; a superuser |
| Redis | `SHELL_REDIS_URL` | No verified TLS; no password |
| Secrets manager | `SHELL_SECRETS_SOURCE` | Secrets from elsewhere |
| Edge | `SHELL_TRUSTED_PROXIES` | No trusted proxy |
| Email | `SHELL_SMTP_URL` | A Mailpit host |
| Audit storage | `SHELL_AUDIT_STORAGE__*` | No independent deletion audit store |

## 6. Data ownership and database layout

Kratos owns identities, credentials, email addresses, sign-in sessions,
flows and codes; Hydra owns clients, keys, grants and refresh tokens. Shell
never touches their databases and keeps no copy of addresses: when it needs
one, it maps the user's UUID to the provider identity ID and asks
`IdentityAdmin`. `shell` holds the tables below and business data; Redis
only disposable counters, locks, jobs and single-use, short-lived codes: the
gates' challenges (trusted-device proofs, every-time sign-ins) and
re-sign-in continuation codes.

**Linking.** `users` and `staff` use our UUID as key, with (`issuer`,
`provider_identity_id`) unique, so the per-request lookup (section 3) is one
indexed query. Subscribers appear on first sight, staff only by script;
nothing links the two.

**Layout.** One PostgreSQL server holds `shell` (roles `shell_app`,
`shell_migrator`, `shell_audit_purge`) and each realm's Kratos and Hydra
databases, a role each, with no superusers or unmanaged extensions; behind a
transaction-mode pooler, asyncpg's statement cache is off. Tables: `users`
(name, `locale`, `timezone`, accepted terms, `require_trusted_devices`,
`disabled_at`), `staff` (with `disabled_at`), `roles`, `staff_roles`,
`devices` (one per sign-in: sign-in ID, user or staff member, the provider's
login session ID, browser and platform, city, signed in, last active),
`trusted_devices` (user, public key, name, platform, enrolled, last used),
`known_browsers` (user or staff member, browser or install ID hashed per
account, first and last seen),
`passkey_labels` (the provider's credential ID and a name, since the
provider names a passkey only when it is added), `pending_changes` (pending
deletions only, each for an account already disabled), `tasks` and
`audit_log` (actor realm and ID or an operator's cloud identity, client and
token ID, action, target, outcome, correlation ID, IP, user agent, time),
which `shell_app` may only insert into and read; entries are purged after
**2 years**. Device records go when their session ends, trusted devices on
removal or expiry, and known browsers after 400 days unseen or with the
account (section 8). Before irreversible deletion, its audit record 
(Shell UUID, provider issuer and identity ID, deletion ID and time) 
is durably written through a port to append-only storage outside the 
databases' restoration boundary. Writes are idempotent; deletion waits 
if the write fails. Records follow the audit log's retention period and 
residency rules.

**Durable work:** `tasks` rows, written in the use case's Unit of Work and
run by arq; a sweep each minute re-enqueues lost jobs, and a daily one only
tidies away expired device records, expired trusted devices and known
browsers unseen for 400 days (section 8). Unfinished device records expire
after ten minutes; the worker schedules their removal at that deadline
and retries after interruption. Expired unfinished records are never
counted, listed or accepted, whether or not cleanup has run.

**Residency:** everything that stores data, backups included, runs in
`SHELL_REGION`; the privacy policy names Google, the email relay and the
browser ID.

## 7. Token and cookie flow

| Cookie | Holds | Attributes |
|---|---|---|
| `__Host-shell-access`, `-refresh`, `-id` | Access, refresh, ID token | `HttpOnly`; Strict |
| `__Host-shell-csrf` | CSRF token | Script-readable; Strict |
| `__Host-shell-signin` | State, verifier, nonce; a re-sign-in's continuation code | `HttpOnly`; Lax; 10 min |
| `__Host-shell-browser` | Random browser ID | `HttpOnly`; Lax; 400 days |

All are `Secure` and `Path=/`, and all but the browser ID expire with their
contents. The admin host sets the same cookies as `__Host-shell-admin-*`.
The sign-in cookie is encrypted, single-use and Lax, since returns via
Google or an off-domain provider are cross-site; the browser ID is Lax
because our consent step reads it after such a return, and 400 days is
Chrome's cap. The ID token is kept only for the hint that sign-out sends.

**Browser (BFF)**

1. `GET /v1/auth/login?return_to=/path` (relative only) stores state,
   nonce, an S256 PKCE verifier and the return path, then redirects for
   `openid`, `offline_access` and the client's scopes on the API audience.
2. After sign-in, Hydra always calls the consent step, which our app runs
   in every environment: it grants first-party (`skip_consent`) clients
   `openid` and `offline_access` at once via `accept_consent`, stamps the
   sign-in ID, creates the device record, writes the sign-in's audit entry
   and checks the browser ID (section 8).
3. `/v1/auth/callback` checks `state`, redeems the code with the client
   assertion and verifier, validates both tokens (the ID token's `nonce`
   included), sets the cookies and redirects via the terms page on the app
   host until it is done (section 8).

**Refresh (BFF)**

1. On 401 `auth.token_expired`, the app calls `POST /v1/auth/refresh`, one
   call per tab at a time.
2. The BFF locks a hash of the refresh token in Redis (5 s wait); a token
   rotated within 30 s gets 409 `auth.refresh_retry`, and the app retries
   with its new cookies; otherwise the BFF redeems the token once and marks
   it rotated.
3. Grace is **0 s**, so reuse revokes the family: the BFF clears the
   cookies, answers 401 `auth.signed_out` and counts it. Timeouts are
   counted, never retried; a provider outage returns 503 and keeps the
   cookies.
4. The token hook refuses a refresh whose device record is gone or past the
   session's limits, and otherwise notes the record's last activity.

**Sign-out and limits.** `POST /v1/auth/logout` revokes the refresh token,
deletes the device record, clears the session's cookies and the app's cached
data but keeps the browser ID and the device key (no `Clear-Site-Data` for
cookies or storage), so the device stays known and trusted, and sends the
browser through Hydra's sign-out step with the ID-token hint. Ours, from
launch, accepts at once via `accept_logout` only sign-outs our clients
start, with that hint, and asks the user to confirm any other request; Hydra
then ends its sign-in session and the Kratos session
(`urls.identity_provider.url`). With the record gone, the per-request check
refuses any copy of the device's access token at once (To verify: the
sign-in ID survives refreshes). The idle limit is the refresh-token
lifetime, which restarts at each rotation; the absolute limit counts from
the session's first sign-in, the device record's sign-in time, since fresh
sign-ins and passkey confirmations move `auth_time`. The spec sets both.

**Mobile**

1. Sign in via the system browser as a public client with S256 PKCE and a
   claimed HTTPS redirect. The sign-in page, fresh sign-ins and passkey
   confirmations, and the provider's settings pages all open in that sheet,
   so every flow matches the web. Apps accept terms through the API using
   `account:write`.
2. Keep the refresh token in the keychain or keystore and the access token
   in memory; call `api.` with bearer. Keep a device key and a random
   install ID in the platform keystore, and report the ID just after each
   sign-in (section 8).
3. Keep one refresh in flight, storing each new refresh token before use. At
   sign-out, call Shell's sign-out route, which deletes the device record
   and ends the provider session, then revoke and delete the refresh token.

**Agents**

1. As itself: client credentials with a signed assertion; no refresh token.
2. For a user: the mobile sequence, with narrower scopes; fresh-sign-in and
   trusted-device checks refuse it (section 4).

## 8. Account management and sign-out

A fresh sign-in is a re-authentication through Hydra with `prompt=login`,
which Kratos runs as a refresh flow offering an email code or Google
(whether Google asks again is To verify, section 15). On the routes Shell's
gate protects, a passkey can stand in: once Shell confirms the device is
trusted and the switch is on, the client adds a flag to the authorization
request, and our sign-in screen offers the passkey if the provider's flow
lists one for the account. The flag only shapes the screen; the gate checks
the device proof itself. In the customer realm our settings hook refuses a
passkey as the latest step on the provider's settings screens, which can't
see the device key. A fresh sign-in issues new tokens under a new sign-in
ID, to which the device record carries over (below), and the token hook
carries the new `auth_time` and `amr`. The spec sets the windows: one for
the sign-in changes below and logging out other devices, which Kratos's
`privileged_session_max_age` matches, so the provider's settings screens
accept no older sign-in than ours; a shorter one for adding a passkey or
enrolling a device, which our settings hook applies to passkeys added on
those screens, from the session Kratos passes it; and a default for risky
actions, which a route may shorten. Everything is audited; ✉ marks a
notification.

| Action | Runs in | Checks |
|---|---|---|
| Add passkey ✉ | Provider settings, refusing hook | Fresh sign-in, shorter window |
| Rename passkey | `passkey_labels` | None |
| Remove passkey ✉ | Provider settings, refusing hook | Fresh sign-in; staff keep their last |
| Link, unlink Google ✉ | Provider settings, refusing hook | Fresh sign-in; linking rules |
| Trust this device ✉ | `trusted_devices` | First risky action with the switch on; fresh sign-in, shorter window |
| Remove a trusted device ✉ | `trusted_devices` | None: it only tightens access |
| Require trusted devices ✉ | `users` | On: none. Off: an enrolled device and a fresh sign-in |
| Name, language, zone | `users` | None |
| Sign out this device | `revoke`; device record | None |
| Log out another device, all | Device records; `revoke_session`, or both revocations for all | Fresh sign-in |
| Delete account ✉ | `pending_changes`; coordinated deletion (section 9) | A sign-in every time, then "Are you sure?" within the spec's time |

The provider's profile settings are off, so addresses change no other way
than the staff script's `change-email`.

**Across sign-in methods.** Each sign-in method finds the account by its own
key: an email code by the address, Google by the Google account's ID, never
by its address. Our entry screen has one email step, which sends a code
either way and signs in an existing account, however it was made, or signs
up a new one; "Continue with Google" sends no code and likewise signs in a
known Google account or signs up a new one. A new Google account signs up
only if Google has verified its address (our claims mapper checks) and no
account uses that address; otherwise our sign-up hook tells the user to use
another sign-in method, after which they can link Google in settings.
Linking happens only there, one Google account per user, under the checks
above. A new account first completes our terms step, whose items (terms of
service, privacy policy and confirmations such as a minimum age) are written
at deployment; until the user accepts the required ones, the API refuses the
account's other calls with 403 `auth.terms_required`. Sign-in pages offer
the email step and Google only; passkeys appear only when signing in again
is due on a trusted device with the switch on, for an account that has one.
The spec lists each flow.

**Trusted devices.** A device's key is a Web Crypto key the page can use but
can't read out or, in apps, a key in the platform keystore; Shell keeps the
public key in `trusted_devices`, since Ory has nothing like it. While the
switch is on, each request to a route with a trusted-device part carries the
device's signature over a fresh single-use challenge, which Shell checks
against the public key, noting the device's last use. Removing a device
revokes it at once, and every check refuses one unused for the spec's period
at the moment of use. Turning the switch on needs nothing and enrolls
nothing; turning it off needs an enrolled device and a fresh sign-in, so a
stolen session can't switch it off. Enrolled devices stay while it is off,
doing nothing and expiring on the same schedule, so turning it back on soon
after needs no new enrollment. "Add a passkey" stays available whatever the
switch, and settings say that passkeys confirm only on trusted devices while
it is on. A fork may force the switch on for every account in configuration.

**The device list.** Our consent step gives each sign-in an ID, put in the
ID token and, by the token hook, in every access token, and creates its
device record (section 6), with the city `GeoLocator` finds for the client
IP. Only the device holding a session can continue it: before a fresh
sign-in or passkey confirmation it gets a one-time continuation code from
Shell with its current session (the BFF keeps it in the sign-in cookie, an
app in memory), and afterwards hands it back with its new session. Shell
then moves the old record's sign-in time to the new record and deletes the
old one, ending the old session, so the list keeps one entry whose sign-in
time is when the session began; if the code never comes back, both entries
stay until their own limits end them. The list shows a record only while its
session is live, judged from its last activity and sign-in time against the
limits, and only once its first tokens are issued.

**New-device notices.** The consent step hashes the browser ID; an app
instead reports its install ID, in the body of a signed-in request, just
after each sign-in. A hash not yet in `known_browsers`, or an app that
reports no known ID within the spec's wait, triggers the notice, which a
task sends. Both realms do this. Shell stores only the hashes, keyed per
account so one browser can't link two accounts.

## 9. Administration

**Request path:** React-admin → edge (`admin.`) → `/v1/admin` with cookies
and CSRF header → guard (workforce issuer, admin audience, scope, `admin`
role, passkey sign-in) → identity use case → repositories and the customer
realm's `IdentityAdmin` → audit entry in the same Unit of Work. Staff list,
view (with recent audit entries), create, edit, disable, enable and delete
subscribers, list and revoke all their sessions and grants, and cancel a
pending deletion. The high-risk ones (disabling, enabling or deleting a
subscriber, ending their sessions, cancelling a deletion) need a passkey
confirmation within the spec's window (section 4), and every support action
notifies the subscriber. The console shows staff their own device list, with
log out one or all (section 8).

**Data section:** each context may register a table for the console,
declaring the fields listed, searched, filtered and editable (no other
field leaves the API), the actions allowed (add, edit, delete, or none for
a read-only table), each action's scope and gate, and its own handlers
where the table has rules. Shell generates the table's routes under
`/v1/admin/<context>/<table>`, behind the guard and in the route catalogue,
writing through the Unit of Work with an audit entry, reads included; the
console builds the table's pages from them, and deletes need `confirm`.
Shell registers `users` (name, language and timezone editable; disabling
and deletion stay with the support actions above) and, read-only, `staff`,
`tasks` and `audit_log`; device, trusted-device, known-browser and
passkey-label rows appear only on a subscriber's page, and the providers'
data never. Rows change here; schemas change only through migrations.

**Coordinated disable:** `DisableUser` sets `disabled_at` and writes the
audit entry and a `tasks` row in one commit, so the API refuses the user at
once; the task runs `disable_user`, `revoke_sessions` and
`revoke_refresh_grants` idempotently, retrying at 1, 5 and 30 minutes, then
hourly for a day, auditing failures, then alerting. Enabling works alike.

**Deletion**, by the user (section 8) or an admin, starts as a coordinated
disable and records the pending deletion. After the spec's delay, the task
claims it, ending the cancellation window, writes its independent audit
record (section 6), sends the completion notice, runs `delete_user` and
removes our rows (fork tables cascade or set null). Support may cancel
before the claim and keep the account disabled: the support freeze.

**Staff lifecycle:** only an operator (cloud IAM, MFA) manages staff, with
`scripts/provision_staff.py`; no page can, the console included. `add`
creates the workforce identity and `staff` row with the `admin` role, plus
a one-time enrollment code, handed over out of band, for registering
passkeys; `re-enroll` issues a new code once the operator has confirmed
who is asking (lost passkeys, and the break-glass path in the spec);
`change-email` replaces the address; `remove` marks the row disabled, so
the API refuses the person at once, then disables the identity and
revokes sessions and grants as a coordinated disable does. Each command
writes an audit entry naming the operator and notifies every admin. The
script warns before removing one of the last two admins, and the console
reminds staff with one passkey to add another. Kratos shows enrollment
codes only while its recovery flow is on, so the workforce realm keeps it
on but gives staff addresses no recovery role: self-service recovery
reaches no one.

**Security:** every admin endpoint needs the guard and CSRF protection and
writes an audit entry, reads included (PIPA's access record); destructive
actions need a confirmation step and a `confirm` field naming the target;
the admin host gets the strictest CSP; provider admin APIs and database
tools stay internal.

## 10. Error model

The domain raises domain errors; adapters translate theirs (provider
timeouts and 5xx into `ProviderUnavailable`, unique violations into
`Conflict`), so no provider or SQL error escapes; one handler maps errors to
status and code, unmapped ones to a bare 500 `internal.unexpected`. Bodies
are RFC 9457 Problem Details: `type` (`urn:shell:error:<code>`), `title`,
`status`, `code`, `correlation_id`. A stale sign-in gets 401
`auth.recent_sign_in_required` and an RFC 9470 challenge, `Bearer
error="insufficient_user_authentication"` with `max_age`. A risky action
from a device that isn't trusted while the switch is on gets its own error,
whose code the spec sets, and the client answers it by trusting this device
(section 8).

Framework errors use the same model: `request.not_found`,
`request.method_not_allowed` and `request.validation_failed`;
unexpected errors use `internal.unexpected`. Required HTTP headers,
such as `Allow`, are preserved.

- **Codes:** `<area>.<condition>` in snake case (`auth.token_expired`),
  listed in OpenAPI, never reused, translated by clients. Statuses: 401
  token, stale sign-in or a device that isn't trusted while the switch is
  on; 403 scope, role, terms or an agent at a gate; 409 conflict or pending
  change; 422, 429 and 503 with `Retry-After`. Shell's authentication
  errors never reveal whether an account exists (Kratos's code sign-in
  can; section 11).
- **Correlation IDs:** the edge overwrites any client-supplied
  `X-Request-ID`; Shell trusts it only from known proxies, then carries
  it through logs, audit entries, errors, responses and provider calls.
- **Timeouts and retries:** provider calls get 1 s to connect, 5 s in all;
  idempotent ones retry twice with jitter; user creation, code redemption
  and refresh never retry. Keys refresh every 5 minutes and on unknown
  `kid`s (at most every 30 s); if fetches fail, the last good keys serve
  1 hour (alert at 15 minutes), then all tokens are refused.
- **Provider down:** sessions last until access tokens expire; refresh
  returns 503 `provider.unavailable`; sign-in shows the edge's
  "unavailable" page; provider-bound admin actions return 503 while queued
  steps wait. Readiness depends on usable cached keys, not provider availability.

## 11. Security controls, rate limiting and threat model

**Baseline controls:** code limits per address and IP (below); notifications
on new credentials, trusted devices and sign-ins from new devices, queued as
`tasks`; user verification required, and checked by the server, and security keys
accepted on every passkey ceremony (section 5); an empty CORS
allowlist, as browsers call only their own origin; both revocation calls for
every account-wide revocation; provider public APIs only via the edge; TLS
on every connection outside development (section 13); HSTS (a year,
subdomains), a per-host CSP with `object-src 'none'`, `base-uri 'none'` and
`frame-ancestors 'none'`, `nosniff`, a strict referrer policy and `no-store` 
on sign-in, token and account responses; the audit log (section 6); pinned, 
scanned images (section 13); `no-reply@` mail via the Workspace relay with 
SPF, DKIM, DMARC at quarantine, and quota alerts.

| Rate-limit layer | Keyed by | Limits (values in the spec) |
|---|---|---|
| Edge | IP (IPv6 by /64) | Flow submissions |
| `EmailSender` | Address | Codes per hour and per day; resend gap |
| `EmailSender` | IP (IPv6 by /64) | Code requests; sign-ups |
| `EmailSender` | Realm | Codes per hour and per day |
| Provider | Sign-in flow | Code lifetime; attempts per flow |

Kratos counts attempts per flow, across resends, and has no resend gap, so
`EmailSender` enforces the gap. The edge sets `True-Client-Ip`, which
Kratos forwards with each email, so `EmailSender` also counts code
requests and sign-ups per IP; the edge, which can't tell requests from
submissions, limits all flow submissions. The realm-wide cap stops a flood from many
addresses from using up the relay's quota, which would stop every code
sign-in. Over-limit codes are dropped with
a 2xx, logged and counted; transient send failures return 5xx so the
courier retries, up to `courier.message_retries`. CAPTCHA, if abuse calls
for it (spec, Later), goes in our sign-in screen or at the edge: Kratos
has it only in Ory's enterprise build.

| Threat | Mitigation | Residual risk |
|---|---|---|
| Mailbox or Google takeover | Notices, device list, support freeze; Google links only in settings | The email ceiling |
| Address passes to someone else (expired domain, reused address) | Google accounts found by ID, not address | The new holder can sign in by code: the email ceiling |
| Stolen device or security key | Device lock; user verification checked by the server (AUTH-PASSKEY-001) | None known |
| Malware on a trusted device | Device lock, short confirmation windows | Can act through the browser itself |
| Stolen or copied session | `HttpOnly`, CSP, short-lived tokens; fresh sign-in before sign-in changes and deletion; with the switch on, an enrolled device and a recent sign-in for risky actions; logged-out devices refused | With the switch off, everything else a signed-in user can do, risky actions included, until logged out |
| CSRF, code injection, mix-up | Strict cookies, PKCE, `nonce` | None known |
| Subscriber to admin | Realm separation | None known |
| Enumeration, code guessing | Limits, generic screens | Code sign-in reveals unknown addresses, and may reveal disabled ones (section 15) |
| Rogue or phished staff | Passkeys, passkey confirmation before high-risk actions, audit, confirmation, alerts on staff changes | Abuse within role |
| Email outage | Queued retries | No code sign-ins or code confirmations; Google sign-ins and passkey confirmations still work |
| Provider outage | Local verification | No new sign-ins |
| Key compromise | Per-realm keys, rotation | Until keys refetch |

**The email ceiling.** A subscriber signs in with an email code or Google,
and signing in again is also how they get back in, so every account is only
as secure as its mailbox or Google account, whose protections Shell can't
see. Passkeys and trusted devices don't raise that ceiling: anyone who can
sign in can add their own. What they stop is a copied or stolen session,
which can't pass a fresh sign-in without the mailbox or a passkey, or, with
the switch on, use a risky route without an enrolled device's key. Shell
accepts the ceiling, softened by notices (sign-ins from new devices
included), the device list, the deletion delay and the support freeze.

## 12. Internationalization

Times are UTC `timestamptz`, sent as ISO 8601 with offsets and shown in
`users.timezone` (IANA). Language is `users.locale` (BCP 47), or before
sign-in the screen's choice, then `Accept-Language`, then English. Web uses
FormatJS catalogues with ICU plurals, admin React-admin's translation
provider; CI fails on missing keys. Clients translate errors by code, our
sign-in module provider messages by numeric ID. `EmailSender` renders
Jinja2 and Babel templates in the recipient's language and timezone or,
before a profile exists, the language our sign-in screen sends in the
flow's transient payload, then the request's `Accept-Language`. Names are
one NFC-normalized field; a new language is catalogues plus one
configuration entry.

## 13. Operations baseline

**Development:** one command starts Compose with Caddy (local CA, edge
limits), `api`, `worker`, PostgreSQL, Redis, each realm's Kratos, Hydra and
reference UI, and Mailpit, bound to `127.0.0.1`, with Ory in production mode
(its HTTPS issuers and Secure cookies served through Caddy's local CA), a
git-ignored `.env` and synthetic seed data. **Deployed:** the same image and
configuration on the compute platform chosen at deployment (README,
Deployment Decisions), with two or more `api` replicas, a `worker`, our
sign-in module, the Workspace relay and the secrets manager, starting empty.
Internal traffic uses TLS under a private CA made for the deployment, its
key in the secrets manager: the deploy job issues and renews each service's
certificate, and for internal hosts clients trust only that CA
(`SHELL_INTERNAL_TLS=private_ca`). A platform that encrypts
service-to-service traffic itself, such as a service mesh, can take its
place (`SHELL_INTERNAL_TLS=platform`).

**CI/CD (GitHub Actions):** lint, types, imports, tests on fakes and
the integration suite → OpenAPI and catalogue checks → image build
and scans (fixable high or critical findings fail). Checks join CI
as their subjects are built and run on every push and pull request.
The deployment job runs migrations (expand, then contract), provider
migrations, ES256 key setup, internal certificates, client registry
and rolling restart.

**Integration suite.** Run on every push and pull request, it proves
each behaviour of Ory, the edge and browsers that our code relies on;
each item's tests arrive with the milestone that builds it (roadmap):

- Access tokens carry `auth_time`, `amr` and the sign-in ID, refreshes
  included, and a failing hook issues none; `sub` is the provider identity
  ID; `amr` lists each sign-in step in order, so its last entry is the
  latest: an email code then a passkey ends in `webauthn`, a passkey then an
  email code in `code`; refreshes keep the latest `auth_time`.
- The sign-in ID survives refreshes; a logged-out device's refresh fails and
  its access tokens are refused; logging out another device ends its
  provider session.
- A passkey re-authenticates a refresh flow on its own, is listed there only
  for an account that has one, and is never offered at sign-in; required AAL
  `aal1` never demands it at sign-in or in settings; a passkey confirmation
  through Hydra issues tokens with a fresh `auth_time` and `webauthn` in
  `amr`.
- Two `api` replicas refreshing at once rotate once; reuse or revocation
  ends the chain; each rotation restarts the idle lifetime.
- Client assertions use ES256, the token URL as `aud` and a fresh `jti`;
  agents use client credentials; first-party clients skip the consent
  screen.
- Both key sets are ES256; a new key signs at once, the old one stays
  published, and no set is ever emptied.
- Codes are single-use, bound to their flow and follow the spec's lifetime
  and per-flow attempt limit; a code sign-in verifies its address.
- Every code reaches `EmailSender` with the identity, language and client
  IP, and failed sends are retried.
- The sign-up and sign-in hooks report each one with its method.
- Settings need a sign-in within the spec's window for sign-in changes, and
  the settings hook receives the session; profile edits are off; passkey and
  Google changes reach our hooks before saving, and in the customer realm
  the hook refuses them when the latest step is a passkey.
- Kratos accepts passkey ceremonies in which our screen requires user
  verification or uses a security key, in both methods (`webauthn`: section
  15).
- Every sign-up and sign-in combination in the spec behaves as listed
  (section 8), and no account is ever linked on a matching address.
- Kratos refuses a Google ID token with a wrong signature, issuer,
  audience, expiry or `nonce`, served by a stand-in answering for Google's
  endpoints (AUTH-IDP-006).
- The customer realm has no recovery flow. An operator-issued code lets
  staff enroll a passkey, in a session that carries the recovery method;
  self-service recovery reaches no staff address; a removed staff member is
  refused at once.
- The provider's own sign-in session ends after the spec's limit, remembered
  sign-ins included.
- Sign-out ends the refresh chain, Hydra's sign-in session and the Kratos
  session.
- The edge enforces its per-IP limit, and only the client IP it sets is
  counted and logged.
- Each realm's host serves screens and provider paths without clashes, and
  Strict token cookies survive a cross-site return in Chromium, Firefox and
  WebKit.

**Mobile integration tests.** The native mobile test clients run against
the real Compose stack in CI, using XCTest on the iOS Simulator and
instrumented tests on the Android Emulator. They and the physical-device
checks use isolated HTTPS test hosts with the platform's required domain
associations.

**Observability:** internal `/healthz` and `/readyz` (database,
compatible schema and, once the provider is integrated, usable signing
keys); JSON logs with correlation ID, realm, user ID, route, status and
duration, never secrets, addresses, or browser or install IDs; metrics on
routes, provider calls, keys, refreshes, emails, limits and tasks, alerting
on errors, stale keys, failed tasks and email quota. The edge logs request
paths without their query strings (NFR-010).

**Backups and keys:** every database, the providers' included, daily with
point-in-time recovery where offered, encrypted and in the region, with one
full restore tested before launch; Hydra's system secret, which encrypts
the signing keys, belongs to the recovery kit. A restore re-applies independent 
deletion audit records before serving traffic, including unfinished irreversible 
deletions, without sending another notice. Replay is idempotent; a failed 
replay or unavailable audit history prevents startup. Secrets list the active
value first, so rotation is prepend, deploy, drop. Signing keys rotate
every 90 days: add an ES256 key, which Hydra signs with at once while
publishing the old one; delete the old key an hour after its last token
expires, or at once on compromise, never emptying a key set (Hydra would
generate an RS256 key, which Shell refuses). The DB-IP City Lite file is
refreshed monthly.

## 14. Import rules

A **package** is `shell_api.shared` or one `shell_api.contexts.<name>`,
each with the layers `domain`, `application`, `infrastructure` and
`presentation`. Each rule is one import-linter contract with external
packages included; rule 2 also forbids indirect imports, the rest check
direct ones. Tests and migrations are exempt. Contracts cover concrete 
package paths, including new contexts; their negative tests exercise 
the same configuration that CI runs.

1. **Layers** (layers contract, one container per package):
   `presentation | infrastructure` (independent) above `application` above
   `domain`.
2. **Pure core** (forbidden): `*.domain` and `*.application` import none of
   `fastapi`, `starlette`, `pydantic`, `pydantic_settings`, `sqlalchemy`,
   `asyncpg`, `alembic`, `redis`, `arq`, `httpx`, `authlib`, `jwt`,
   `jinja2`, `babel`, `maxminddb`.
3. **Web** (forbidden): only `*.presentation` and `shell_api.core` import
   `fastapi` or `starlette`; only those and `*.infrastructure` import
   `pydantic`.
4. **Persistence** (forbidden): only `*.infrastructure` imports
   `sqlalchemy` or `asyncpg`; only `shell_api.shared.infrastructure` and
   `shell_api.core` import `redis` or `arq`.
5. **Libraries** (forbidden): only
   `shell_api.contexts.identity.infrastructure.idp` imports `authlib` or
   `jwt`; only `*.infrastructure` imports `httpx`; only
   `shell_api.shared.infrastructure.email` imports `jinja2` or `babel`;
   only `shell_api.contexts.identity.infrastructure.geo`, the `GeoLocator`
   adapter, imports `maxminddb`.
6. **Vendor code** (forbidden): apart from their submodules, only
   `shell_api.core` imports `shell_api.contexts.identity.infrastructure.idp.ory`
   or `shell_api.contexts.identity.presentation.ory`.
7. **Contexts** (independence over `shell_api.contexts.*`): a context
   imports another only through `<other>.application.api`.
8. **Shared** (forbidden): `shell_api.shared` imports no
   `shell_api.contexts` or `shell_api.core`.
9. **Composition root** (forbidden): only `shell_api.main` and
   `shell_api.worker` import `shell_api.core`.

## 15. To verify before launch

Only these can't be settled in advance; each is checked by hand once its
subject exists (roadmap):

- Settings hooks can tell adding a passkey or Google from removing one.
- The provider doesn't reveal a disabled account before its code is entered;
  if it does, that is accepted with the enumeration risk (section 11).
- The sign-in ID our consent step stamps survives every refresh, and the
  token hook runs at each one. If either fails, logging out another device
  does what logging out all devices does (both revocations), so the device
  is always out, and the user signs in again on their own. Token checks
  (AUTH-TOKEN-003) are never relaxed.
- The provider's own linking of a sign-in method at sign-in can be turned
  off (AUTH-SIGN-009).
- A passkey confirmation through Hydra works on its own: Kratos's refresh
  flow can be started for the account, lists a passkey only for an account
  that has one, accepts it while our screen requires user verification and
  keeps required AAL `aal1`; the tokens carry a fresh `auth_time` and an
  `amr` ending in `webauthn`; and the client's flag reaches our screen.
- Google can be made to ask again for a fresh sign-in; until then, Google
  users sign in again by email code.
- Device keys: Web Crypto keys survive restarts and updates in Safari,
  Chrome and Firefox; what Safari's deletion of a site's script-writable
  storage, after seven days of Safari use without interaction on the site,
  does to them; private windows; the app keystores.
- Devices and browsers: passkey confirmations and staff passkey sign-in
  (autofill, a phone through the cross-device flow, a security key) on iOS,
  Android, macOS and Windows; code entry on phones; app sign-in through the
  system browser; Google sign-in in Safari, Chrome and Firefox.
- Load on sign-in, refresh and token verification at expected peak: on the
  default platform for the baseline, and on its chosen platform for each
  fork's deployment.
- The provider enforces user verification in every passkey ceremony.
  Where it does not, Shell's server-side check cannot be bypassed
  through either the browser or API flow (section 5).

The hands-on rehearsals the spec relies on are done the same way before
launch: a backup restore that re-applies later deletions, a signing-key
rotation (section 13), break-glass re-enrollment and a WCAG 2.2 AA review.

## 16. Later

Project-wide items are in the README's
[Planned Evolution](../README.md#planned-evolution). Specific to this file:

| Later | Trigger |
|---|---|
| Refresh grace of a few seconds | Unexpected sign-outs become a problem |
| DPoP-bound mobile and agent tokens | Device token theft, or a partner asks |
| Immediate revocation of an access token whose device stays signed in, or of a service token | Such a token must stop working before it expires |
| Per-user or per-client quotas | Authenticated abuse, or paid tiers |
| A minimum mobile app version | A fix must outrun the deprecation window |
| Admin host behind a VPN or allowlist | Fixed staff networks, or compliance |
| Automated signing-key rotation | Several deployments |
| Generic answers for unknown addresses | Enumeration abuse, or Kratos covers code sign-in |
| Passkey confirmation inside the mobile app, without the browser sheet | The sheet proves clumsy |
| Device Bound Session Credentials for trusted browsers | The browsers a fork's users use support them |
| A per-route option that always needs a trusted device, even with the switch off | A fork needs device binding on some actions without forcing the switch on for every account |
| Exporting records from the console's Data section, under its own rules | Support or a regulator needs bulk data |
| Right-to-left layouts | A right-to-left language |
| Translated business content | A project needs it |
| Circuit breaking on identity-provider calls | Provider outages that cascade |
| Separate database instance for a self-hosted identity provider | Load or isolation needs |
| Dedicated transactional email provider | The realm-wide code cap is reached in normal traffic, or the relay's limits or deliverability become a constraint |

## 17. Open questions

None.
