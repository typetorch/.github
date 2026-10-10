# typetorch.dev login: one Roblox sign-in for every backend

Status: spec, agreed 2026-10-10. For the backend agent and for whoever builds typetorch.dev. The host name below is
`typetorch.dev` throughout; if the service ends up on typetorch.app, replace the name, nothing else changes.

**For the backend agent.** This is a trial first. Work on the branch `central-oauth` of the backend repo, never on
`main`. Keep every change behind `TYPETORCH_CENTRAL_LOGIN` (off by default on this branch until the trial is
reviewed), add the tests listed under "Backend changes", and leave the admin token and per-game Roblox sign-in
untouched. Use a fake issuer in tests; do not call a real typetorch.dev. Report what you could not verify.
The broker itself is built in the repository `typetorch/dash` from `plans/dash-agent-prompt.md`; that file fixes
the exact JSON of `/report` (canonical JSON `{fingerprint, origin, label, iat}` signed by the instance key, the
202 challenge answer, the hour-long bearer token), `/authorize`, `/token` and the JWKS. Build to those shapes.
For an end-to-end test, run dash locally (`bun run dev` and `bun run fake-roblox` in its repo) and point the
backend at it with `TYPETORCH_CENTRAL_LOGIN_ISSUER=http://127.0.0.1:8788` and the fake's client id and JWKS.

## Summary of the flow

1. Once per project: on typetorch.dev the owner signs in with Roblox, clicks Add project and pastes the fingerprint
   that `typetorch init` printed. The backend has reported its origin and passed the challenge, so the entry links
   to it.
2. Every login: the project link opens the backend's `/auth/typetorch/start`, which makes `state`, `nonce` and a
   PKCE verifier and redirects to typetorch.dev; typetorch.dev redirects to Roblox with the backend's nonce; Roblox
   redirects back to typetorch.dev; typetorch.dev (after a one-time interstitial per project) redirects to the
   backend's callback with a single-use code; the backend redeems the code server to server and gets typetorch.dev's
   assertion plus Roblox's id token, verifies both, and decides the role from its own access list.
3. Once per device: the owner blesses the browser with `typetorch backend bless` (signed by the prod signing key),
   the admin token, or a passkey. Blessed device plus central login is full admin; unblessed is `web` or refused.

What this guarantees: typetorch.dev cannot log anyone in on its own, a backend cannot affect another project, and a
full compromise of typetorch.dev yields a list of users and projects, an outage of this one login method, and at
worst a read-only foothold during a victim's live login. It cannot reach admin, deploys or game data.

## Goal

A user with several games signs in with Roblox once, at typetorch.dev, and from then on every TypeTorch backend they
own or may view accepts that identity. Nobody has to create a Roblox OAuth app per game. The existing logins stay:
the admin token and the per-game Sign in with Roblox keep working unchanged, and a backend operator can switch the
central login off.

typetorch.dev is an identity broker, nothing more. It holds one Roblox OAuth app, confirms who a person is, and
hands a signed statement of that to the backend the person is signing into. It never decides what the person may do
there, never holds backend secrets, and never holds Roblox tokens beyond the sign-in itself.

## Roles

| Party | Has | Decides |
|---|---|---|
| Roblox | the Roblox OAuth app owned by TypeTorch | that the person controls the Roblox account |
| typetorch.dev | a session cookie per signed-in person; an Ed25519 signing key pair; a list of backends each person has signed into | that this browser session belongs to Roblox user N |
| a backend | its **instance key** (an Ed25519 pair made on first start, kept in the data dir; its fingerprint is the project's identity), its access list (owners from the signed settings record, `TYPETORCH_WEB_VIEWERS`) and its own session store | the role: owner, web (read-only) or refused |

## Project identity: the fingerprint

A backend's identity on typetorch.dev is the fingerprint of its instance key, printed as `tt1-<base32 of the first
20 bytes of sha256(public key)>`. A URL is not an identity: a quick-tunnel backend has a new origin on every start.

- `typetorch init` prints the fingerprint at the end of the backend phase, and `typetorch doctor` shows it. On
  typetorch.dev the person clicks **Add project**, pastes it and gives it a label. The entry exists before the
  backend was ever reached.
- **The backend reports its origin, and proves it.** On every start, every tunnel URL change and once a day, it
  POSTs `https://typetorch.dev/report` with `{ "fingerprint", "origin", "label", "iat" }` and a signature over that
  JSON by the instance key, plus the public key. typetorch.dev verifies the signature and that `iat` is within five
  minutes. When the origin is new for this project it runs a challenge, in the manner of ACME but on the backend's
  own API path: it answers `{ "challenge": "<token>" }`, the backend serves the token at
  `<origin>/api/typetorch/challenge/<token>` for 60 seconds, typetorch.dev fetches it over https (no redirects, 5 s)
  and compares. Only then is the origin stored as the project's current address. The answer carries a short-lived
  access token (one hour, bound to the fingerprint) that later reports in that hour may use instead of a signature
  and challenge; it is derived, never stored by the backend past its life, and never the root of anything. A
  report with a bad signature or a failed challenge is dropped and counted. The quick-tunnel wrapper
  (`typetorch backend run`) triggers the report after each `backend setup`.
- No DNS records, no names under typetorch.dev, no certificates: the project page links to the current origin,
  which is all a stable bookmark needs.
- **The assertion's audience is the fingerprint**, and the backend compares it with its own key, so
  `TYPETORCH_PUBLIC_URL` may change freely. typetorch.dev sends the code only to the origin the backend last
  reported, and `redirect_uri` must be on that origin.
- The interstitial is per person and fingerprint, so it shows once, and the project list never collects stale
  entries. A project whose last report is older than 30 days is shown greyed, not deleted.
- The instance key is not a secret of the game: losing it means a new fingerprint and a new Add project. It is
  backed up with the data dir like everything else there.

## The flow

Relying-party initiated, authorization code with PKCE, no client secret. The backend is the relying party.

1. The person opens the backend and clicks **Sign in with typetorch.dev** (or lands on `/login` from the typetorch.dev
   project list; that link carries nothing but the backend's address).
2. The backend creates `state` (32 random bytes), `nonce` (32 random bytes) and a PKCE `code_verifier` (43 to 128
   characters), stores all three in a short-lived, HttpOnly, SameSite=Lax cookie bound to this login attempt (or in
   its session store), and redirects to

   ```
   https://typetorch.dev/authorize?response_type=code
     &redirect_uri=https://backend.example.com/auth/typetorch/callback
     &state=<state>&nonce=<nonce>
     &code_challenge=<base64url(sha256(code_verifier))>&code_challenge_method=S256
   ```

   plus `&project=<fingerprint>`. There is no `client_id`: the relying party is the project, and typetorch.dev
   requires `redirect_uri` to be on the origin that project last reported.
3. typetorch.dev checks `redirect_uri`: absolute, `https` (plain `http` only for `localhost` and `127.0.0.1`), no
   fragment, no userinfo, path under the origin, and on the project's last reported origin. It then sends the
   person through Roblox OAuth (scopes `openid profile` only) **with the backend's `nonce` as the OIDC nonce**, on
   every login, even when a typetorch.dev session exists (Roblox auto-approves an app the person already consented
   to, so this costs one redirect). On the first sign-in to this project for this person it shows an interstitial:
   "Sign in to <label> (tt1-...) as <Roblox name>?" with Continue and Cancel. Later sign-ins to the same project
   skip it.
4. typetorch.dev issues a code: 32 random bytes, stored server side with `{ aud: fingerprint, redirect_uri, sub:
   robloxUserId, nonce, code_challenge, roblox_id_token, expires: now + 60 s, used: false }`, and redirects to
   `redirect_uri?code=<code>&state=<state>`.
5. The backend's callback checks `state` against its cookie, then POSTs server to server:

   ```
   POST https://typetorch.dev/token
   { "grant_type": "authorization_code", "code": "<code>",
     "redirect_uri": "https://backend.example.com/auth/typetorch/callback",
     "code_verifier": "<code_verifier>" }
   ```

6. typetorch.dev checks: the code exists, is unused and unexpired; `redirect_uri` matches byte for byte;
   `sha256(code_verifier)` matches the stored challenge. It marks the code used, records the backend origin in the
   person's project list, and answers with an **identity assertion**:

   ```
   { "assertion": "<compact JWS>", "roblox_id_token": "<the id token Roblox issued for this login>",
     "sub": "409950512", "name": "phasenull", "display_name": "phasenull" }
   ```

   The JWS header is `{ "alg": "EdDSA", "kid": "<key id>" }`. The payload is

   ```
   { "iss": "https://typetorch.dev", "aud": "tt1-<fingerprint>", "sub": "409950512",
     "name": "phasenull", "display_name": "phasenull", "nonce": "<nonce>",
     "iat": 1760000000, "exp": 1760000120, "jti": "<unique id>" }
   ```

7. The backend verifies the signature against typetorch.dev's published keys
   (`GET https://typetorch.dev/.well-known/jwks.json`, cached by `kid`, refreshed on an unknown `kid`), then checks
   `iss`, `aud` equals its own fingerprint, `nonce` equals the cookie's, `exp` is in the future, `iat` is at most
   two minutes old, and `jti` was not seen before (a small in-memory set with expiry).

   **Then it verifies Roblox's own token**, so that typetorch.dev alone can never produce a login: the
   `roblox_id_token` signature against Roblox's published keys (its OIDC discovery document, cached), `iss` is
   Roblox, `aud` is TypeTorch's Roblox client id (a public constant in the backend, overridable by env), `nonce`
   equals the cookie's nonce, `sub` equals the assertion's `sub`, and it is unexpired. Either check failing is "not
   signed in". A failed response from `/token` is treated the same; the response body is never shown to the person.
8. The backend maps `sub` to a role with the same code path as per-game Roblox sign-in today: an owner from the
   access list, a user id in `TYPETORCH_WEB_VIEWERS` as a viewer, anyone else refused with the same page as a wrong
   token. **Admin needs a blessed device**: an owner on a device that was blessed once (see "Trust concentration")
   gets full admin; an owner on any other device gets what `TYPETORCH_CENTRAL_LOGIN_UNBLESSED` says, `web`
   (read-only, the default) or `refuse`. It then creates a fresh session
   (never reuse a pre-login session id), with the same lifetime rules as the other logins, and records
   `login: typetorch.dev` in the audit log.

Logout is local to the backend. typetorch.dev has its own sign-out on its site. There is no single logout.

## Trust concentration: what typetorch.dev's operator can and cannot do

The limit, stated plainly: a broker in the browser flow can hijack a login that is happening at that moment. While a
person signs in, typetorch.dev controls the Roblox authorize URL in their browser and could put a nonce from a session
the operator started into it. Roblox cannot bind its token to the final backend, so no signature scheme removes this.
Everything else is removed, and the live hijack is made loud and nearly worthless:

1. **Roblox's id token passes through with the backend's nonce inside** (steps 3, 6, 7). typetorch.dev cannot sign
   as Roblox, so it cannot mint a login for anyone at any time. The only thing it could do is redirect a login the
   victim is performing right then, and the victim's own login then fails visibly, which the backend logs as
   `nonce mismatch after typetorch.dev login`.
2. **Admin needs a device blessed by something the broker never sees.** Blessing is done once per browser and
   sets a device cookie (`tt_device`, 180 days, `Secure`, `HttpOnly`, `SameSite=Lax`, rotated on every use, bound
   to a server-side record that the owner can list and revoke on the Settings page). From then on every typetorch.dev
   login on that device is **full admin, with write access**. A hijacked login lands on the operator's device, which
   was never blessed, so it gets `web` or is refused (`TYPETORCH_CENTRAL_LOGIN_UNBLESSED=web|refuse`). Ways to
   bless, any one of them:
   - **A signed link from the CLI** (preferred): `typetorch backend bless` fetches a one-time challenge from the
     backend, signs it with the game's prod signing key (the root of trust in TypeTorch, on the owner's PC), and
     opens `<backend>/auth/bless?...` in the browser. The backend verifies the signature against the public keys in
     its signed settings record and sets the cookie. Nothing is typed, nothing passes through typetorch.dev, and
     only a holder of the signing key can do it.
   - **The admin token** entered once on that device (`POST /auth/device`).
   - **A passkey** registered at the backend (WebAuthn; static-URL installs only, since the relying party id is the
     host).
   Why this is the only shape that works: the backend must demand something the operator's browser cannot present,
   and Roblox's id token carries nothing from the person's browser except the nonce, which the broker chooses. So
   the extra factor has to arrive by a channel outside the browser flow. The signing key already is one.
3. **The blast radius of a hijack is a `web` session on an unblessed device**, during a victim's live login, with a
   failed sign-in on the victim's screen and a line in the log. Deploys, rollbacks and access changes still need the
   prod signing keys on the owner's PC, which no backend has.
4. **The rest:** signed assertions, optional `kid` pinning, every central login in the audit log with the provider
   and the role granted, the off switch, and the admin token and per-game Roblox sign-in as paths that never touch
   typetorch.dev.

What this buys: without typetorch.dev's cooperation nobody logs in through it; with its cooperation, the operator
can never become admin, because no device of theirs was ever blessed. Full isolation stays one setting away.

## What typetorch.dev stores

Per person: Roblox user id, the name and display name at the last sign-in, the session (cookie id, created, last
seen), and the project list: `{ fingerprint, label, added, last_login }`. Per project: `{ fingerprint, public key,
current origin, label from the last report, last report at }`, where labels are untrusted text escaped on display.
The person can delete entries. Codes live 60 seconds and are deleted on use or expiry. Roblox id tokens are kept
only inside a pending code and go with it.

Not stored: Roblox access or refresh tokens past the sign-in request, backend keys, admin tokens, anything about
players. The Roblox OAuth app asks for `openid profile` only.

## Keys

- One active Ed25519 signing key, one "next" key published alongside it in the JWKS a week before it becomes
  active, so cached key sets never miss. Old keys stay published for a day after their last use.
- The private key lives only on typetorch.dev, in an environment secret, never in the repo.
- A backend may pin the issuer's key ids with `TYPETORCH_CENTRAL_LOGIN_KIDS=<kid>,<kid>`; then an assertion from any
  other key is refused even if the JWKS lists it. Optional, for operators who want it.

## Backend changes

- The instance key: made on first start in `<data dir>/instance.key` (mode 0600), its fingerprint on the startup
  line, in `/healthz` for admins, in `doctor` and at the end of `typetorch init`.
- The report: `POST <issuer>/report` on start, after every `backend setup` (the quick-tunnel wrapper), and daily;
  signed with the instance key; failures logged once an hour, never fatal.
- New env `TYPETORCH_CENTRAL_LOGIN`: `on` (default when `TYPETORCH_PUBLIC_URL` is set), `off` removes the button,
  refuses the callback and sends no reports. `TYPETORCH_CENTRAL_LOGIN_ISSUER` defaults to `https://typetorch.dev`
  (for a self-hosted broker or tests). `TYPETORCH_CENTRAL_LOGIN_UNBLESSED`: `web` (default) or `refuse`, the role
  of an owner on a device that was never blessed.
  `TYPETORCH_ROBLOX_BROKER_CLIENT_ID`: TypeTorch's Roblox client id, a built-in constant, overridable for tests.
- Routes: `GET /auth/typetorch/start` (step 2), `GET /auth/typetorch/callback` (steps 5 to 8),
  `POST /auth/device` (the admin token once, sets the device cookie), `GET /auth/bless/challenge` and
  `GET /auth/bless` (the CLI's signed link), `GET /api/typetorch/challenge/<token>` (the origin challenge, answers
  only while a report is pending), and on static-URL installs the passkey register and assert routes. The Settings
  page lists blessed devices with a revoke button. All login routes are rate limited like the token login and
  counted toward the five-failure lockout.
- The explorer's login page shows three ways when they are enabled: the admin token, Sign in with Roblox (per game),
  Sign in with typetorch.dev. The Settings page gets a read-only line that says whether the central login is on.
- Startup never blocks on typetorch.dev. The JWKS is fetched on the first callback and cached; a fetch failure fails
  that login only.
- The role decision, session creation and audit log reuse the per-game Roblox sign-in code. Only the identity source
  is new.
- Tests: a fake issuer in the test suite (a local key pair, a `/token` and a JWKS route) covers: a good login for an
  owner, for a viewer, for a stranger; a reused code; a wrong `state`; a wrong `nonce`; an `aud` for another origin;
  an expired assertion; an unknown `kid`; a pinned `kid` mismatch; the switch set to `off`; a Roblox id token with
  the wrong nonce, the wrong `aud`, a bad signature or a `sub` that differs from the assertion; an owner getting
  `web` without a device cookie and admin with one; the report's signature and `iat` window.

## typetorch.dev endpoints

| Route | What |
|---|---|
| `GET /authorize` | the checks in step 3, the Roblox round trip with the backend's nonce, the interstitial, the redirect with the code |
| `POST /report` | a backend's signed origin report; verifies the signature and the `iat` window, runs the origin challenge for a new origin, updates the project, returns the hour-long access token |
| `POST /token` | step 6; answers 400 with a short error code (`invalid_grant`, `invalid_request`) and no detail |
| `GET /.well-known/jwks.json` | the public keys, `Cache-Control: max-age=3600` |
| `GET /` | the project list for a signed-in person: **Add project** (paste a fingerprint, give a label), each entry a link to `<current origin>/auth/typetorch/start`, greyed when the last report is older than 30 days; delete per entry; sign out |
| `GET /auth/roblox/*` | its own Roblox OAuth, standard code flow with PKCE and `state` |

Rate limits: `/authorize` and `/token` per address and per origin; one pending code per `(sub, origin)` at a time.
Every error page is static text with no reflected input.

## Security checklist

The property each line protects, so a reviewer can tick them:

- **A code is useless to anyone but the backend that asked for it.** PKCE verifier, exact `redirect_uri` match,
  60 second life, single use.
- **An assertion is useless to any other backend.** `aud` is the fingerprint; the backend compares it with its own
  instance key, never with a URL or the request's Host header.
- **typetorch.dev cannot log anyone in on its own.** Roblox's id token with the backend's nonce is required, and
  typetorch.dev cannot sign as Roblox.
- **typetorch.dev's operator can never become admin.** Admin needs a device blessed by the signing key, the admin
  token or a passkey, none of which pass through typetorch.dev.
- **Nobody can repoint a project at a phishing origin.** The current origin only changes on a report signed by the
  instance key and confirmed by the challenge served from that origin.
- **A link cannot log a victim into an attacker's account.** The flow starts at the backend with `state` in a cookie
  and `nonce` in the assertion; typetorch.dev never starts a login into a backend by itself.
- **typetorch.dev cannot grant roles.** It sends only `sub` and names; the backend's access list decides.
- **typetorch.dev is not an open redirector.** `https` only, origin checks, the first-visit interstitial.
- **A replayed assertion fails.** `jti` seen set, `exp`, `iat` window.
- **A leaked signing key has a bounded blast radius.** Rotation through the JWKS, optional `kid` pinning, every
  central login in the backend's audit log, the switch to turn it off.
- **typetorch.dev being down costs nothing but this button.** The admin token and per-game Roblox sign-in do not
  touch it; no startup dependency.
- **typetorch.dev is a poor target.** It stores no secrets of value: user ids, names, origins, sessions.
- **No mixed content, no downgrade.** Cookies `Secure`, `HttpOnly`, `SameSite=Lax`; HSTS on typetorch.dev; a
  callback over plain http is refused except on loopback.

## Accepted limits

- During a person's live login, a malicious typetorch.dev could redirect that one login to a session of its own,
  on an unblessed device, and the person sees their own login fail. There is no scheme that
  removes this while a broker is in the browser flow; per-game Roblox sign-in and the token avoid the broker.
- A backend that never reports (central login off, or typetorch.dev unreachable) cannot be reached from the project
  list, and its logins through typetorch.dev fail until a report lands.
- Nothing stops a backend operator from reading who signed in through typetorch.dev; that is the backend's own log.
- Roblox may require review of the TypeTorch OAuth app once it has more than a handful of users. To verify before
  launch: Roblox's current limit on unreviewed OAuth apps and the review process.

## Out of scope

Accounts, email, passwords, billing, DNS names or certificates under typetorch.dev, per-user settings on typetorch.dev, a list of a person's games taken from
Roblox (the project list comes only from sign-ins that happened), and any API through which typetorch.dev would reach
into a backend.
