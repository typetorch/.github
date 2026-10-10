# `typetorch init`: the guided installer

Status: plan, agreed 2026-10-10. Lives here until the cli repo takes it into `plans/`.

Goal: someone with no backend or Roblox tooling knowledge runs one command and ends with a live game that hot-swaps,
a self-hosted backend if they want one, and a prompt for a coding agent to build the game itself. The installer
replaces steps 1 to 11 of `docs/getting-started/fresh-setup.md` and the backend's Coolify and VPS sections with
questions, actions and checks. The long guides stay as the reference each step links to.

No hosted TypeTorch service is involved. Tunnels run in the user's own Cloudflare account and zone. No accounts, no
email, no sign-ups.

## 1. Shape

- One command in `@typetorch/cli`: `bunx @typetorch/cli init` in an empty folder, or `bun run typetorch init` in a game.
- A wizard of phases. Each phase is ask, do, check. The check reuses the probes `doctor` already runs, so the wizard
  never reports success on something `doctor` would flag.
- Resumable. Progress goes to `.typetorch/init.json` (never a secret). Re-running continues at the first unfinished
  phase. An already-configured game gets a repair menu that lists what is missing or broken.
- Non-interactive: `--answers <file>` replays a JSON of answers for tests and agents. Secrets still come only from
  `.env` or the environment.
- Every step prints one line on why it exists and the doc section it replaces.

## 2. TUI primitives

The CLI has no runtime dependencies and keeps none. Add `src/tui.ts` on top of `interact.ts`:

- `select` (arrow keys, number keys as a fallback), `text` with a validator, `secret` (masked), `confirm`,
- `step` header with the "why" line and link, `checklist` that updates lines in place, a spinner on the existing
  progress line,
- works in PowerShell, cmd, macOS Terminal and Linux terminals. Tests use `scriptedInteraction`.

About 400 lines. If hand-rolling turns out to cost more than that, `@clack/prompts` is the one dependency allowed.

## 3. Phases

1. **Preflight.** Detect OS, Bun, git, Rokit. Offer to install Bun and Rokit with the official install scripts, after
   a y/N that shows the command. git on Windows: the winget line. Detect the folder: empty means the template path; an
   existing roblox-ts project means "this is a migration", print the agent prompt and stop; an existing
   `typetorch.json` means repair mode.
2. **Project.** Name, clone the template, drop its origin, `bun install`, `rokit install`, first `typetorch build`.
   Failures print the fix.
3. **Roblox.** The user pastes the experience URL from Creator Hub; the wizard parses the universe and place ids.
   Then the Open Cloud key, masked, with the credentials URL and the scope table printed. The key is probed at once
   and the universe fetched through Open Cloud, so the creator (group or user) is filled in without a question. The
   owner's Roblox username is resolved to a user id for `members`. Shows `typetorch.json` before writing it, writes
   `.env`.
4. **Signing keys.** `keys init` and `keys init --fallback`, then the key file paths and a confirm that they are backed
   up before moving on.
5. **Kernel place.** "Is this place empty?" Empty: `kernel deploy --replace-place`. Has content: the patch engine,
   after checking the Save Place API setting and that Team Create is closed. Check: join the game, `/tt status`.
6. **Backend.** Section 4.
7. **First deploy and dev branch.** Commit, `deploy` on main, wait for the user to join, create `dev`, deploy it,
   `access push`.
8. **Agent handoff.** Section 5.

## 4. Backend phase

Three questions, each shown only when an earlier answer needs it.

### Q1. How will the backend run?

| Answer | Meaning |
|---|---|
| Coolify | The Docker Compose app from the backend repo, behind Coolify's proxy |
| Linux service | systemd unit plus the files in the backend's `server/` folder, no Docker |
| This PC | Windows or macOS: the backend runs under Bun as a login task (scheduled task or launchd) |
| Skip for now | Nothing written. The summary lists what is lost: alerts, auto rollback, analytics |

### Q2. How do we reach the machine? (Coolify and Linux service)

| Answer | Meaning |
|---|---|
| SSH | Every command runs on the VPS over one SSH session from this PC |
| Local | You are already on the VPS; commands run here with sudo |

SSH details: host, user (default `root`), port (default 22), then auth: an SSH key file (default `~/.ssh/id_ed25519`,
with a list of `~/.ssh`) or a password. A password is typed into the real `ssh` prompt, never put on a command line
or in a file. To type it once, the wizard uploads one bash script per phase and runs it, instead of many commands.
The first action is a test connection that prints the remote OS and whether sudo works.

Local details: sudo available, Debian or Ubuntu, nothing listening on 80, 443 or 8787.

### Q3. How is it exposed?

| Answer | Meaning |
|---|---|
| Cloudflare tunnel with a static hostname (recommended) | A named tunnel in the user's own Cloudflare account, a fixed `sub.their-domain` |
| Public IPv4 and my own domain | Caddy with Let's Encrypt, the DNS A record checked against the machine's public address |
| Quick tunnel (tests only) | `trycloudflare.com`, the URL changes on every start |

**Cloudflare tunnel, static hostname.** Needs a domain on Cloudflare (free plan is enough) and an API token with
Account: Cloudflare Tunnel: Edit and Zone: DNS: Edit, scoped to that zone. The wizard:

1. asks for the token, masked, with the exact token template to create in the Cloudflare dashboard;
2. lists the zones the token can see, the user picks one and a subdomain (default `typetorch`);
3. creates the tunnel (`POST /accounts/{id}/cfd_tunnel`, `config_src: cloudflare`), sets the ingress (the backend's
   address, then a `http_status:404` catch-all), creates the proxied CNAME to `<tunnel-id>.cfargotunnel.com`;
4. installs cloudflared on the machine and runs `cloudflared service install <tunnel token>`;
5. does not store the Cloudflare API token anywhere. Teardown or rotation asks for it again. The tunnel token is kept
   by cloudflared's own service config, not by the game's `.env`.

The backend then runs with `TYPETORCH_CLOUDFLARE=on` and `TYPETORCH_PUBLIC_URL=https://sub.their-domain`, bound to
loopback behind the connector. The hostname never changes: reconnects, reboots and a reinstall with the same token
keep it. On Coolify the connector points at Traefik on `localhost:80` and the app's domain is the hostname; this is
the layout Coolify's own tunnel guide uses and needs one verification on a real Coolify box.

Limits that matter, from Cloudflare's docs: 1,000 tunnels per account, 200 DNS records on a free zone created after
September 2024 (3,500 on Pro), 100 MB request bodies, a 125 s proxy read timeout. One game uses one tunnel and one
record, so none of them is a concern for a user's own zone. The explorer's live streams must send keepalives under
125 s.

**Quick tunnel.** Kept only for test games with no domain. The wizard wraps the backend's `bun run local`, which
re-runs `backend setup` on every start because the URL changes. The summary says it dies with the PC.

### After every path

The two 32-character keys are generated on this PC, written to the game's `.env`, and reach the machine only inside
the env file over SSH or in the local env file. Then `backend setup --url https://<hostname>` runs. The check is
`typetorch servers` after the user joins the game. An optional last question takes a Discord or Slack webhook URL for
alerts.

**Explorer exposure.** With a tunnel the admin UI is public on that hostname. The wizard defaults
`TYPETORCH_TOKEN_LOGIN=off` with Roblox sign-in on, or sets the admin allow list to the user's current address,
and explains the lockout after five failed logins.

### Per-run specifics

- **Linux service:** swap, ufw, Bun under `/opt/bun`, the `typetorch` service user, clone to
  `/opt/typetorch-backend`, `bun install` and the explorer build, `/etc/typetorch/backend.env` with mode 640, the
  unit, then Caddy or cloudflared per Q3, then `GET /healthz` through the public URL. One checklist line per step.
  Idempotent: a rerun skips what exists. `--teardown` reverses it.
- **Coolify:** "Is Coolify installed?" No: run its install script, tell the user to open the panel and set the admin
  account. Then print the exact resource settings (Docker Compose, repo URL, compose location, domain) and the env
  block, wait for Enter, probe `/healthz` and both keys. Coolify is clicked by the user; the wizard prepares and
  verifies.
- **This PC:** install Bun if missing, clone the backend next to the game, build the explorer, write a local env
  file, register a login task that runs the server, then Q3 (tunnel recommended; a public IPv4 on a home PC is
  discouraged and the wizard says why).

## 5. Agent handoff

The last phase asks two lines: "what is your game?" and "what should the first version do?". The wizard writes
`AGENT_PROMPT.md` into the repo: what init set up, the ids, the branch map, the rules from `docs/agents/AGENTS.md`,
and the two answers. It prints the prompt and, if `claude` is on PATH, offers to run `claude` with it and the
typetorch plugin marketplace added. The agent works locally only, per the playbook, and keeps the "what you need to
do" list current.

## 6. Rules the wizard obeys

- Secrets are masked on input, never printed, never written to `init.json` or `AGENT_PROMPT.md`.
- Every file write is shown first. Every external action (publish, deploy, SSH, Cloudflare API) has its own y/N.
- Ctrl+C anywhere leaves a consistent state; the next `init` resumes.
- No GitHub Actions, no hosted CI, nothing leaves the user's machine except to Roblox, their VPS and their Cloudflare
  account.

## 7. Milestones

| Milestone | Scope | Result |
|---|---|---|
| M1 | `tui.ts`, phases 1 to 5 and 7 | Zero to a live hot-swap with no backend and no doc reading |
| M2 | Phase 6: skip, this PC, Coolify, with quick tunnel and static tunnel | A backend for test games and for people on Coolify |
| M3 | Phase 6: Linux service over SSH and local, Caddy path, `--teardown` | A full self-hosted backend from one prompt |
| M4 | Phase 8, repair mode, docs rewrite | fresh-setup.md becomes "run init"; AGENTS.md moves init from Planned to Today |

## 8. Known limits

- Studio and the browser cannot be skipped: creating the experience, the Open Cloud key and the Cloudflare token
  happen there. The wizard points at the exact page and verifies the result.
- The SSH installer owns a server. It must be idempotent and have `--teardown`, or a half-failed run leaves a VPS the
  user does not understand.
- Password SSH works but is the slow path; the wizard steers to a key file.

## 9. Files

In `cli/src`: `tui.ts`, `commands/init.ts` (the phases), `init/` with one file per phase
(`preflight.ts`, `project.ts`, `roblox.ts`, `keys.ts`, `kernel.ts`, `backend.ts`, `deploy.ts`, `agent.ts`),
`init/ssh.ts` (the session, script upload), `init/cloudflare.ts` (zones, tunnel, DNS), `init/state.ts`
(`.typetorch/init.json`). Backend repo: `server/` gains `install.sh` (the Linux service steps as one idempotent
script the wizard uploads) and `cloudflared.service` notes. Docs: fresh-setup.md and fleet-and-alerts.md point at
init first.
