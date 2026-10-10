# Prompt for the `dash` agent

Copy everything below the line into the first message of an agent working in an empty repository named
`typetorch/dash`. It is self-contained: the agent does not need the other plan files.

---

You are building **dash**, the service that runs at `https://typetorch.dev`: one Roblox sign-in for every TypeTorch
backend. A TypeTorch backend is a self-hosted server that game owners run (repository `typetorch/backend`); dash
lets a person sign in with Roblox once and then sign in to each of their backends with one click, without creating a
Roblox OAuth app per game. dash is an identity broker and a bookmark list, nothing more. It never decides what a
person may do on a backend, never holds backend secrets, never holds Roblox tokens beyond a sign-in, and never
proxies any traffic.

This repository starts empty. Build it from scratch to the spec below, on the branch `main`. The backend side is
being built at the same time by another agent on the backend repo's `central-oauth` branch, against the same
contract, so the JSON shapes and routes below are fixed: do not rename or reshape them. Where the spec is silent,
choose the simplest thing and write it in the README.

## Rules

1. **Never create, ask for, read, print or store a secret** that is not yours to generate: no Roblox client secret
   in the repo, no `.env` committed, no example file with real values. Secrets come from the environment.
2. **No GitHub Actions, no hosted CI.** Tests run locally with `bun test`. Do not add `.github/workflows`.
3. **Runtime: Bun 1.3+, TypeScript, no framework.** `Bun.serve`, `bun:sqlite`, `node:crypto` (Ed25519 through
   `crypto.sign` / `crypto.verify` and `generateKeyPairSync("ed25519")`). Avoid dependencies; if one is unavoidable,
   say why in the README. Ships as one Docker image that works as a Coolify "Docker Compose" app, like
   `typetorch/backend` does: a root `Dockerfile`, a `compose.yaml` with a data volume at `/data`, a health check on
   `/healthz`, port 8787 exposed, the container as a non-root user.
4. **Every input is untrusted.** Labels from backends are escaped on display. Error pages are static text with no
   reflected input. `/token` answers 400 with a short code and no detail.
5. **Write tests for every security property listed at the end** before you call anything done. Use a fake Roblox
   (a local server with its own key pair, discovery document, authorize, token and JWKS routes) and a fake backend
   (a relying party that runs the full flow) in the test suite.
6. Keep a README with: what dash is, the flow, every route with its JSON, every environment variable, how to run
   locally against a backend on the `central-oauth` branch, how to deploy on Coolify, and what dash stores.

## Terms

- **Backend**: a TypeTorch backend instance. Identified by its **fingerprint**: `tt1-` followed by the base32
  (lowercase, no padding) of the first 20 bytes of `sha256(raw Ed25519 public key)` of its **instance key**.
- **Project**: dash's record of a backend: fingerprint, public key, current origin, label, last report time.
- **Person**: a Roblox user signed in to dash.
- **Origin**: `https://host[:port]`, no path. Plain `http` is allowed only for `localhost` and `127.0.0.1`.

## Roblox sign-in (dash's own)

dash owns one Roblox OAuth app. Standard OIDC authorization code flow with PKCE and `state`, scopes `openid profile`.
Discovery at `https://apis.roblox.com/oauth/.well-known/openid-configuration` (configurable, for the fake Roblox in
tests). Verify the id token's signature against Roblox's JWKS, `iss`, `aud` equals our client id, `nonce`, `exp`.
Store nothing from Roblox except `sub` (the user id), `preferred_username` and `name` at this sign-in. Do not store
access or refresh tokens.

Session: a random 32-byte id in a cookie `dash_session`, `Secure`, `HttpOnly`, `SameSite=Lax`, host-only (no
`Domain` attribute), 12 hours idle, 7 days max. Rows in SQLite.

## Routes

### `GET /` (signed in)

The project list for this person: label, fingerprint, current origin, last report time, greyed when the last report
is older than 30 days. Each entry is a link to `<current origin>/auth/typetorch/start`. **Add project**: a form with
a fingerprint and a label; the fingerprint must match `^tt1-[a-z2-7]{32}$`; adding an unknown fingerprint is allowed
(the backend may report later) and shows "waiting for the backend's first report". Delete per entry. Sign out.
Signed out: a page with one Sign in with Roblox button.

### `POST /report` (from a backend)

Body:

```json
{ "fingerprint": "tt1-...", "public_key": "<base64 raw Ed25519 public key>",
  "origin": "https://backend.example.com", "label": "Target Rush", "iat": 1760000000,
  "signature": "<base64 Ed25519 signature>" }
```

The signature is over the canonical JSON of `{ fingerprint, origin, label, iat }` with keys in that order and no
whitespace. Checks: the fingerprint matches the public key; the signature verifies; `iat` within 5 minutes of now;
`origin` is a valid origin; `label` at most 64 characters. Alternatively the request may carry
`Authorization: Bearer <access token>` from an earlier report in the last hour instead of `public_key` and
`signature`; the token is bound to the fingerprint.

If the origin equals the stored one: update `last_report`, answer 200 with a fresh access token. If the origin is
new for this project: run the **challenge**. Generate a 32-byte token, answer

```json
{ "challenge": "<token>", "retry_after": 2 }
```

with status 202, and within 2 to 10 seconds fetch `<origin>/api/typetorch/challenge/<token>` over https, no
redirects, 5 second timeout, at most 3 attempts. The body must be exactly the token. On success store the origin
and `last_report`, and on the backend's next report (it polls with the same body, or the bearer token) answer 200:

```json
{ "ok": true, "origin": "https://backend.example.com", "access_token": "<token>", "expires_in": 3600 }
```

On failure the origin is not stored; the next report starts a new challenge. A report with a bad signature or a
bad fingerprint is 401 and counted per address. Rate limit: 60 reports per fingerprint per hour, 600 per address.

### `GET /authorize` (from a person's browser, started by a backend)

Query: `response_type=code`, `project=<fingerprint>`, `redirect_uri`, `state`, `nonce`, `code_challenge`,
`code_challenge_method=S256`. Checks, in order, each failing with a static error page:

1. the project exists and has a stored origin;
2. `redirect_uri` is absolute, on that origin exactly (scheme, host, port), has a path, no fragment, no userinfo;
3. `state`, `nonce` (16 to 128 characters), `code_challenge` (43 to 128 characters) present.

Then **always** send the person through Roblox, even with a dash session, with `nonce` set to the backend's `nonce`
(Roblox auto-approves an app the person already consented to). Keep the pending authorization in a server-side row
keyed by a cookie-bound id. When Roblox returns, verify the id token as above and that its `nonce` equals the
backend's, and keep the raw id token in the pending row.

If this person has never continued to this project before, show the interstitial: "Sign in to <label>
(tt1-...) as <name>?" with Continue and Cancel; remember Continue per (person, project). Then create the code: 32
random bytes, row `{ code, fingerprint, redirect_uri, sub, name, display_name, nonce, code_challenge,
roblox_id_token, expires: now + 60 s, used: false }`, and redirect to `redirect_uri?code=<code>&state=<state>`. One
pending code per (sub, fingerprint): a new one replaces the old.

### `POST /token` (from a backend, server to server)

```json
{ "grant_type": "authorization_code", "code": "<code>",
  "redirect_uri": "https://backend.example.com/auth/typetorch/callback",
  "code_verifier": "<verifier>" }
```

Checks: the code exists, unused, unexpired; `redirect_uri` equals the stored one byte for byte;
`base64url(sha256(code_verifier))` equals the stored challenge. Mark used (a second use is 400 `invalid_grant`,
and the first session is not revoked: log it). Answer:

```json
{ "assertion": "<compact JWS>", "roblox_id_token": "<the raw id token from this login>",
  "sub": "409950512", "name": "phasenull", "display_name": "phasenull" }
```

The assertion: header `{ "alg": "EdDSA", "kid": "<key id>", "typ": "JWT" }`, payload

```json
{ "iss": "https://typetorch.dev", "aud": "tt1-<fingerprint>", "sub": "409950512",
  "name": "phasenull", "display_name": "phasenull", "nonce": "<the backend's nonce>",
  "iat": 1760000000, "exp": 1760000120, "jti": "<16 random bytes, base64url>" }
```

`iss` is the configured public URL. Errors: 400 `{ "error": "invalid_grant" }` or `{ "error": "invalid_request" }`,
nothing else in the body. Rate limit per address.

### `GET /.well-known/jwks.json`

The active signing key and the next one, each `{ "kty": "OKP", "crv": "Ed25519", "kid", "x", "use": "sig",
"alg": "EdDSA" }`. `Cache-Control: public, max-age=3600`. Keys that signed anything in the last 24 hours stay listed.

### `GET /healthz`

`{ "ok": true }`. Never anything secret.

### `GET /auth/roblox/start`, `GET /auth/roblox/callback`, `POST /auth/signout`

dash's own Roblox sign-in and sign-out.

## Keys

Ed25519 signing keys live in `<data dir>/keys/`: `active.key`, optional `next.key`, and `retired/<kid>.key` kept for
24 hours. `kid` is the first 8 bytes of `sha256(public key)` as hex. First start generates `active.key`. A CLI
subcommand `bun run keys rotate` makes `next.key` if missing, or promotes `next` to `active` and retires the old
one. Files are mode 0600, never logged, never served.

## Data (SQLite, `<data dir>/dash.sqlite`, WAL)

- `people(sub PRIMARY KEY, name, display_name, created, last_login)`
- `sessions(id PRIMARY KEY, sub, created, last_seen)`
- `projects(fingerprint PRIMARY KEY, public_key, origin, label, first_report, last_report)`
- `bookmarks(sub, fingerprint, label, added, last_login, continued, PRIMARY KEY (sub, fingerprint))`
- `pending_auth(id PRIMARY KEY, sub NULL, fingerprint, redirect_uri, state, nonce, code_challenge, roblox_id_token
  NULL, created)`
- `codes(code PRIMARY KEY, fingerprint, redirect_uri, sub, name, display_name, nonce, code_challenge,
  roblox_id_token, expires, used)`
- `challenges(fingerprint, origin, token, created, PRIMARY KEY (fingerprint))`
- `report_tokens(token_hash PRIMARY KEY, fingerprint, expires)`
- `rate(key PRIMARY KEY, window_start, count)`

A sweeper every minute deletes expired codes, challenges older than 10 minutes, pending auths older than 15 minutes,
report tokens past expiry, and sessions past their limits. Store token and session ids hashed where they are
bearer secrets (`report_tokens`, `sessions`).

## Environment

| Variable | Required | What |
|---|---|---|
| `DASH_PUBLIC_URL` | yes | `https://typetorch.dev`; the `iss`, the Roblox redirect base, cookie `Secure` |
| `ROBLOX_OAUTH_CLIENT_ID`, `ROBLOX_OAUTH_CLIENT_SECRET` | yes | dash's Roblox app |
| `ROBLOX_OIDC_DISCOVERY` | no | defaults to Roblox's discovery URL; the fake Roblox in tests |
| `DASH_DATA_DIR` | no | `/data` in Docker, `./data` otherwise |
| `PORT`, `HOST` | no | 8787; `127.0.0.1`, `0.0.0.0` in Docker |
| `DASH_TRUST_PROXY` | no | `1` behind Coolify's proxy: client addresses from `X-Forwarded-For` |
| `DASH_ALLOW_HTTP_ORIGINS` | no | `1` allows plain-http origins beyond loopback, for local testing only; the startup line warns |

The server refuses to start without the required values. The startup line says what is set, never the values.

## Local development against a backend

Document and support: `bun run dev` starts dash on `http://127.0.0.1:8788` with `DASH_PUBLIC_URL` set to that and
`ROBLOX_OIDC_DISCOVERY` pointing at the fake Roblox started by `bun run fake-roblox` (it signs in any user id you
type). The backend on `central-oauth` runs with `TYPETORCH_CENTRAL_LOGIN_ISSUER=http://127.0.0.1:8788`,
`TYPETORCH_ROBLOX_BROKER_CLIENT_ID=<the fake client id>` and its Roblox JWKS pointed at the fake. Write the exact
commands for both sides in the README.

## Security properties to test

Each one is a test, named for the property:

- a code redeemed with the wrong `code_verifier`, the wrong `redirect_uri`, after 60 s, or a second time, fails;
- a code for project A is refused when redeemed with project B's redirect URI;
- `/authorize` refuses a `redirect_uri` on any origin but the project's stored one, a `http` origin that is not
  loopback, a fragment, userinfo, an unknown project, a project without an origin;
- the assertion's `aud` is the fingerprint, `nonce` is the backend's, `exp` is 120 s after `iat`, `jti` is unique,
  the signature verifies with the JWKS and fails after one byte changes;
- the Roblox id token returned by `/token` carries the backend's nonce and is the one from this login, not an
  earlier one;
- a report with a bad signature, a mismatched fingerprint, a stale `iat`, or a label over 64 characters is refused
  and changes nothing;
- a report with a new origin stores nothing until the challenge body matches; a wrong body, a redirect, a timeout
  and a 404 all leave the old origin;
- a bearer report token works for its fingerprint only and not after an hour;
- a person who never continued the interstitial for a project is shown it; one who did is not;
- deleting a bookmark removes it for that person only;
- key rotation: an assertion signed by the retired key verifies for 24 hours, then the key is gone from the JWKS;
- rate limits answer 429 and never leak which limit;
- `/healthz`, error pages and logs contain no token, code, secret or id token;
- cookies are `Secure`, `HttpOnly`, `SameSite=Lax` and host-only.

## Done means

`bun test` green with the tests above, `bun run dev` plus `bun run fake-roblox` lets a backend on `central-oauth`
complete a login end to end (write down what you ran and what you saw), the Docker image builds, `compose.yaml` and
the README are complete, and a short `CHANGELOG.md` says what exists. Finish with a numbered list of what the user
must do by hand: create the Roblox OAuth app at Creator Hub with the redirect `https://typetorch.dev/auth/roblox/callback`,
put the two values in Coolify, set `DASH_PUBLIC_URL` and `DASH_TRUST_PROXY=1`, check Roblox's review requirements
for an OAuth app used by many people, and deploy.
