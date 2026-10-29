# Prompt for the `dash` agent

Copy everything below the line into the first message of an agent working in an empty repository named
`typetorch/dash`. It is self-contained: the agent does not need the other plan files.

---

You are building **dash**, the service that runs at `https://dash.typetorch.dev`: one Roblox sign-in for every
TypeTorch backend. A TypeTorch backend is a self-hosted server that game owners run (repository `typetorch/backend`);
dash lets a person sign in with Roblox once and then sign in to each of their backends with one click, without
creating a Roblox OAuth app per game. dash is an identity broker and a bookmark list, nothing more. It never decides
what a person may do on a backend, never holds backend secrets, never holds Roblox tokens beyond a sign-in, and never
proxies any traffic.

This repository starts empty. Build it from scratch to the spec below, on the branch `main`. The backend side is
being built at the same time by another agent on the backend repo's `central-oauth` branch, against the same
contract, so the JSON shapes and routes below are fixed: do not rename or reshape them. Where the spec is silent,
choose the simplest thing and write it in the README.

## Rules

1. **Never create, ask for, read, print or store a secret** that is not yours to generate: no Roblox client secret
   in the repo, no `.env` or `.dev.vars` committed, no example file with real values. Secrets come from Worker
   secrets (`wrangler secret put`) or a local gitignored `.dev.vars`.
2. **No GitHub Actions, no hosted CI.** Tests run locally with `bun test`. Do not add `.github/workflows`.
3. **Runtime: a Cloudflare Worker, TypeScript, no framework.** The Worker's `fetch` and `scheduled` handlers, D1 (the
   `DB` binding) for storage, Web Crypto (`crypto.subtle`) for Ed25519, SHA-256 and random bytes. Deployed with
   wrangler: `wrangler.toml` with a D1 binding, the cron trigger, and the `dash.typetorch.dev` custom domain. No
   runtime dependencies; development dependencies are `wrangler`, `typescript` and `@types/bun`. If a runtime
   dependency is unavoidable, say why in the README. `/healthz` stays.
4. **Every input is untrusted.** Labels from backends are escaped on display. Error pages are static text with no
   reflected input. `/token` answers 400 with a short code and no detail.
5. **Write tests for every security property listed at the end** before you call anything done. Use a fake Roblox
   (a local server with its own key pair, discovery document, authorize, token and JWKS routes) and a fake backend
   (a relying party that runs the full flow) in the test suite. Tests run against an in-memory D1 shim over
   `bun:sqlite` that applies the real migrations.
6. Keep a README with: what dash is, the flow, every route with its JSON, every environment variable, how to run
   locally against a backend on the `central-oauth` branch, how to deploy with wrangler, and what dash stores.

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
`Domain` attribute), 12 hours idle, 7 days max. Rows in D1, with the id stored as a hash.

The client address for rate limits is the `CF-Connecting-IP` header. Cloudflare sets it and clients cannot forge it.

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

The challenge fetch runs after the 202 answer (Workers `ctx.waitUntil`), not in the request.

### `GET /authorize` (from a person's browser, started by a backend)

Query: `response_type=code`, `project=<fingerprint>`, `redirect_uri`, `state`, `nonce`, `code_challenge`,
`code_challenge_method=S256`. Checks, in order, each failing with a static error page:

1. the project exists and has a stored origin;
2. `redirect_uri` is absolute, on that origin exactly (scheme, host, port), has a path, no fragment, no userinfo;
3. `state`, `nonce` (16 to 128 characters), `code_challenge` (43 to 128 characters) present.

Then **always** send the person through Roblox, even with a dash session, with `nonce` set to the backend's `nonce`
(Roblox auto-approves an app the person already consented to). Keep the pending authorization in a D1 row
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
{ "iss": "https://dash.typetorch.dev", "aud": "tt1-<fingerprint>", "sub": "409950512",
  "name": "phasenull", "display_name": "phasenull", "nonce": "<the backend's nonce>",
  "iat": 1760000000, "exp": 1760000120, "jti": "<16 random bytes, base64url>" }
```

`iss` is the configured public URL (`DASH_PUBLIC_URL`). Errors: 400 `{ "error": "invalid_grant" }` or
`{ "error": "invalid_request" }`, nothing else in the body. Rate limit per address.

### `GET /.well-known/jwks.json`

The active signing key and the next one, each `{ "kty": "OKP", "crv": "Ed25519", "kid", "x", "use": "sig",
"alg": "EdDSA" }`. `Cache-Control: public, max-age=3600`. Keys that signed anything in the last 24 hours stay listed.

### `GET /.well-known/typetorch-login`

Public login metadata, so backends need no Roblox client id of their own:

```json
{ "issuer": "https://dash.typetorch.dev", "jwks_uri": "https://dash.typetorch.dev/.well-known/jwks.json",
  "roblox_client_id": "<ROBLOX_OAUTH_CLIENT_ID>",
  "roblox_discovery": "https://apis.roblox.com/oauth/.well-known/openid-configuration" }
```

`issuer` is `DASH_PUBLIC_URL`; `roblox_discovery` is `ROBLOX_OIDC_DISCOVERY` or Roblox's default. No secret is in
it. `Cache-Control: public, max-age=3600`.

### `GET /healthz`

`{ "ok": true }`. `503 { "ok": false }` when the configuration is missing. Never anything secret.

### `GET /auth/roblox/start`, `GET /auth/roblox/callback`, `POST /auth/signout`

dash's own Roblox sign-in and sign-out.

## Keys

On Workers the private keys are Worker secrets, not files. `DASH_SIGNING_KEY` holds the active Ed25519 private key
as a JWK JSON string (`{"kty":"OKP","crv":"Ed25519","d":"...","x":"..."}`). `DASH_SIGNING_KEY_NEXT` is optional and
holds the next key, published in the JWKS ahead of rotation. `kid` is the first 8 bytes of `sha256(public key)` as
hex. dash refuses to serve (503) without `DASH_SIGNING_KEY`.

`bun run keys` keeps the operator's copy in `./keys` (gitignored, mode 0600 where the OS has modes) and never prints
a private key:

- `bun run keys init` makes `keys/active.key` if missing.
- `bun run keys rotate` makes `keys/next.key` if missing; otherwise it promotes next to active and moves the old
  active key to `keys/retired/<kid>.key`, deleted after 24 hours.
- `bun run keys status` prints the key ids.
- `bun run keys push` runs `wrangler secret put` for `DASH_SIGNING_KEY` (and `DASH_SIGNING_KEY_NEXT` if present),
  through stdin.

Retired keys need no secret. Every signature records the key's public half and the time it last signed in D1
(`signing_keys`), and the JWKS keeps listing any key that signed in the last 24 hours. Private keys are never logged
and never served.

## Data (D1, `migrations/0001_init.sql`)

The schema in `migrations/0001_init.sql` is the source of truth. It has these tables:

- `people(sub PRIMARY KEY, name, display_name, created, last_login)`
- `sessions(id PRIMARY KEY, sub, created, last_seen)`
- `projects(fingerprint PRIMARY KEY, public_key, origin, label, first_report, last_report)`
- `bookmarks(sub, fingerprint, label, added, last_login, continued, PRIMARY KEY (sub, fingerprint))`
- `pending_auth(id PRIMARY KEY, kind, sub NULL, name, display_name, fingerprint, redirect_uri, state, nonce,
  code_challenge, roblox_state, roblox_verifier, roblox_id_token NULL, csrf, created)`
- `codes(code PRIMARY KEY, fingerprint, redirect_uri, sub, name, display_name, nonce, code_challenge,
  roblox_id_token, expires, used)`
- `challenges(fingerprint PRIMARY KEY, origin, token, created, status)`
- `report_tokens(token_hash PRIMARY KEY, fingerprint, expires)`
- `rate(key PRIMARY KEY, window_start, count)`
- `signing_keys(kid PRIMARY KEY, x, last_used)`: public halves only

A cron trigger (`* * * * *` in `wrangler.toml`) runs the sweeper every minute. It deletes expired codes, challenges
older than 10 minutes, pending auths older than 15 minutes, report tokens past expiry, and sessions past their
limits. Store token and session ids hashed where they are bearer secrets (`report_tokens`, `sessions`), and store
codes and pending ids hashed too.

## Environment

Set as wrangler `[vars]` (public values) or `wrangler secret put` (secrets). Local values go in `.dev.vars`
(gitignored). Never commit values.

| Variable | Kind | Required | What |
|---|---|---|---|
| `DASH_PUBLIC_URL` | var | yes | `https://dash.typetorch.dev`; the assertion `iss`, the Roblox redirect base (`<url>/auth/roblox/callback`), cookies get `Secure` when it is https |
| `ROBLOX_OAUTH_CLIENT_ID` | var | yes | dash's Roblox OAuth app (public); published at `/.well-known/typetorch-login` |
| `ROBLOX_OAUTH_CLIENT_SECRET` | **secret** | yes | dash's Roblox OAuth app secret |
| `DASH_SIGNING_KEY` | **secret** | yes | the active Ed25519 private key as a JWK JSON string (`bun run keys init` makes one) |
| `DASH_SIGNING_KEY_NEXT` | **secret** | no | the next key, published in the JWKS ahead of rotation |
| `ROBLOX_OIDC_DISCOVERY` | var | no | defaults to `https://apis.roblox.com/oauth/.well-known/openid-configuration`; the fake Roblox in dev and tests |
| `DASH_ALLOW_HTTP_ORIGINS` | var | no | `1` accepts plain-http origins beyond loopback, for local testing only; the startup line warns |
| `DB` | D1 binding | yes | the D1 database, in `wrangler.toml` |

dash answers 503 (a static page) without the required values. The first request in each isolate logs one startup
line that names what is set, never a value.

## Local development against a backend

Document and support: `bun run dev` runs `wrangler dev` on `http://127.0.0.1:8788` with `DASH_PUBLIC_URL` set to
that, `ROBLOX_OIDC_DISCOVERY` pointing at the fake Roblox started by `bun run fake-roblox` (it signs in any user id
you type, on `http://127.0.0.1:8790`), a dev-only signing key, and a local D1 with the migrations applied. The
backend on `central-oauth` runs with `TYPETORCH_CENTRAL_LOGIN=on` and `TYPETORCH_CENTRAL_LOGIN_ISSUER=http://127.0.0.1:8788`.
It reads the Roblox client id and discovery URL from `http://127.0.0.1:8788/.well-known/typetorch-login`; no Roblox
settings go on the backend. Write the exact commands for both sides in the README.

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
- `/.well-known/typetorch-login` returns the issuer, JWKS URI, client id and discovery URL, and no secret;
- rate limits answer 429 and never leak which limit;
- `/healthz`, error pages and logs contain no token, code, secret or id token;
- cookies are `Secure`, `HttpOnly`, `SameSite=Lax` and host-only.

## Done means

`bun test` green with the tests above, and `bun run typecheck` clean. `bun run dev` plus `bun run fake-roblox` lets a
backend on `central-oauth` complete a login end to end (write down what you ran and what you saw). `wrangler deploy
--dry-run` builds. `wrangler.toml`, the migrations and the README are complete, and a short `CHANGELOG.md` says what
exists. Finish with a numbered list of what the user must do by hand:

1. Create the Roblox OAuth app at Creator Hub with the redirect `https://dash.typetorch.dev/auth/roblox/callback`,
   scopes `openid profile`.
2. Put the client id in `wrangler.toml` `[vars]`, and set the secret with `wrangler secret put ROBLOX_OAUTH_CLIENT_SECRET`.
3. Create the D1 database (`wrangler d1 create dash`), put its id in `wrangler.toml`, and apply the migrations with
   `--remote`.
4. Make and push the signing key (`bun run keys init && bun run keys push`).
5. Check Roblox's review requirements for an OAuth app used by many people.
6. `wrangler deploy`, attach the `dash.typetorch.dev` custom domain, and check `https://dash.typetorch.dev/healthz`.
