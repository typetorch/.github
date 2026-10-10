# typetorch.dev login: one Roblox sign-in for every backend

Status: spec, 2026-10-10. For the backend agent and for whoever builds typetorch.dev. The host name below is
`typetorch.dev` throughout; if the service ends up on typetorch.app, replace the name, nothing else changes.

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
| a backend | its access list (owners from the signed settings record, `TYPETORCH_WEB_VIEWERS`) and its own session store | the role: owner, web (read-only) or refused |

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

   There is no `client_id`: the relying party is identified by the origin of `redirect_uri`.
3. typetorch.dev checks `redirect_uri`: absolute, `https` (plain `http` only for `localhost` and `127.0.0.1`), no
   fragment, no userinfo, path under the origin. If the person has no typetorch.dev session, it sends them through
   Roblox OAuth (scopes `openid profile` only) and back. On the first sign-in to this origin for this person it shows
   an interstitial: "Sign in to backend.example.com as <Roblox name>?" with Continue and Cancel. Later sign-ins to the
   same origin skip it.
4. typetorch.dev issues a code: 32 random bytes, stored server side with `{ aud: origin, redirect_uri, sub: robloxUserId,
   nonce, code_challenge, expires: now + 60 s, used: false }`, and redirects to `redirect_uri?code=<code>&state=<state>`.
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
   { "assertion": "<compact JWS>", "sub": "409950512", "name": "phasenull", "display_name": "phasenull" }
   ```

   The JWS header is `{ "alg": "EdDSA", "kid": "<key id>" }`. The payload is

   ```
   { "iss": "https://typetorch.dev", "aud": "https://backend.example.com", "sub": "409950512",
     "name": "phasenull", "display_name": "phasenull", "nonce": "<nonce>",
     "iat": 1760000000, "exp": 1760000120, "jti": "<unique id>" }
   ```

7. The backend verifies the signature against typetorch.dev's published keys
   (`GET https://typetorch.dev/.well-known/jwks.json`, cached by `kid`, refreshed on an unknown `kid`), then checks
   `iss`, `aud` equals its own public origin (`TYPETORCH_PUBLIC_URL`), `nonce` equals the cookie's, `exp` is in the
   future, `iat` is at most two minutes old, and `jti` was not seen before (a small in-memory set with expiry).
   A failed response from `/token` is treated as "not signed in"; the response body is never shown to the person.
8. The backend maps `sub` to a role with the same code path as per-game Roblox sign-in today: an owner from the
   access list is an admin, a user id in `TYPETORCH_WEB_VIEWERS` is a viewer, anyone else is refused with the same
   page as a wrong token. It then creates a fresh session (never reuse a pre-login session id), with the same
   lifetime rules as the other logins, and records `login: typetorch.dev` in the audit log.

Logout is local to the backend. typetorch.dev has its own sign-out on its site. There is no single logout.

## What typetorch.dev stores

Per person: Roblox user id, the name and display name at the last sign-in, the session (cookie id, created, last
seen), and the project list: `{ origin, first_login, last_login, label }` where `label` is an optional short name the
backend may send with the token request (`"label": "Target Rush"`), treated as untrusted text and escaped on display.
The person can delete entries. Codes live 60 seconds and are deleted on use or expiry.

Not stored: Roblox access or refresh tokens past the sign-in request, backend keys, admin tokens, anything about
players. The Roblox OAuth app asks for `openid profile` only.

## Keys

- One active Ed25519 signing key, one "next" key published alongside it in the JWKS a week before it becomes
  active, so cached key sets never miss. Old keys stay published for a day after their last use.
- The private key lives only on typetorch.dev, in an environment secret, never in the repo.
- A backend may pin the issuer's key ids with `TYPETORCH_CENTRAL_LOGIN_KIDS=<kid>,<kid>`; then an assertion from any
  other key is refused even if the JWKS lists it. Optional, for operators who want it.

## Backend changes

- New env `TYPETORCH_CENTRAL_LOGIN`: `on` (default when `TYPETORCH_PUBLIC_URL` is set), `off` removes the button and
  refuses the callback. `TYPETORCH_CENTRAL_LOGIN_ISSUER` defaults to `https://typetorch.dev` (for a self-hosted broker
  or tests). A backend without `TYPETORCH_PUBLIC_URL` cannot use it: `aud` has nothing to match.
- Routes: `GET /auth/typetorch/start` (step 2), `GET /auth/typetorch/callback` (steps 5 to 8). Both are rate limited
  like the login route and count toward the five-failure lockout.
- The explorer's login page shows three ways when they are enabled: the admin token, Sign in with Roblox (per game),
  Sign in with typetorch.dev. The Settings page gets a read-only line that says whether the central login is on.
- Startup never blocks on typetorch.dev. The JWKS is fetched on the first callback and cached; a fetch failure fails
  that login only.
- The role decision, session creation and audit log reuse the per-game Roblox sign-in code. Only the identity source
  is new.
- Tests: a fake issuer in the test suite (a local key pair, a `/token` and a JWKS route) covers: a good login for an
  owner, for a viewer, for a stranger; a reused code; a wrong `state`; a wrong `nonce`; an `aud` for another origin;
  an expired assertion; an unknown `kid`; a pinned `kid` mismatch; the switch set to `off`.

## typetorch.dev endpoints

| Route | What |
|---|---|
| `GET /authorize` | the checks in step 3, the Roblox round trip, the interstitial, the redirect with the code |
| `POST /token` | step 6; answers 400 with a short error code (`invalid_grant`, `invalid_request`) and no detail |
| `GET /.well-known/jwks.json` | the public keys, `Cache-Control: max-age=3600` |
| `GET /` | the project list for a signed-in person, each entry a link to `<origin>/auth/typetorch/start`; delete per entry; sign out |
| `GET /auth/roblox/*` | its own Roblox OAuth, standard code flow with PKCE and `state` |

Rate limits: `/authorize` and `/token` per address and per origin; one pending code per `(sub, origin)` at a time.
Every error page is static text with no reflected input.

## Security checklist

The property each line protects, so a reviewer can tick them:

- **A code is useless to anyone but the backend that asked for it.** PKCE verifier, exact `redirect_uri` match,
  60 second life, single use.
- **An assertion is useless to any other backend.** `aud` is the origin; the backend compares it with its own public
  URL, not with the request's Host header.
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

- A backend behind a quick tunnel has a new origin on every start, so the interstitial shows again and the project
  list collects stale entries. The list lets the person delete them, and an entry whose origin has not been used
  for 30 days is dropped.
- Nothing stops a backend operator from reading who signed in through typetorch.dev; that is the backend's own log.
- Roblox may require review of the TypeTorch OAuth app once it has more than a handful of users. To verify before
  launch: Roblox's current limit on unreviewed OAuth apps and the review process.

## Out of scope

Accounts, email, passwords, billing, per-user settings on typetorch.dev, a list of a person's games taken from
Roblox (the project list comes only from sign-ins that happened), and any API through which typetorch.dev would reach
into a backend.
