# `typetorch init`: the guided installer

Status: plan, agreed 2026-10-10. Lives here until the cli repo takes it into `plans/`.

Goal: someone with no backend or Roblox tooling knowledge runs one command and ends with a live game that hot-swaps,
a self-hosted backend if they want one, and a prompt for a coding agent to build the game itself. The installer
replaces steps 1 to 11 of `docs/getting-started/fresh-setup.md` and the backend's Coolify and VPS sections with
questions, actions and checks. The long guides stay as the reference each step links to.

No hosted TypeTorch service is involved. No accounts, no email, no sign-ups, no API tokens for anything but
Roblox Open Cloud: a PC gets a quick tunnel the wizard manages, a VPS gets a static address.

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

Asked only where there is a choice. The rule: no login anywhere, nothing to sign up for, and the user never has to
know a URL.

| Where the backend runs | Exposure | Static? |
|---|---|---|
| A PC (Windows, macOS, Linux) | A Cloudflare quick tunnel (`trycloudflare.com`) started by the wizard's wrapper | No, and it does not matter (below) |
| A VPS without a domain | Caddy with a certificate for `backend.<ipv4>.sslip.io` | Yes, the IPv4 is static |
| A VPS with a domain | Caddy with Let's Encrypt, the A record checked against the machine's address | Yes |

A VPS user is asked one question: "do you have a domain pointed at this server?". A PC user is asked nothing.

**Quick tunnel without friction.** The tunnel URL changes on every start, so the wizard never makes the user handle
it. It installs `typetorch backend run`, a wrapper that starts the backend and `cloudflared tunnel --url
http://127.0.0.1:8787` together, reads the new URL from cloudflared's output, runs `backend setup --url <it>` (which
signs the URL into the settings record and pings the servers) and prints the explorer address. Running servers
switch within seconds; the kernel's fleet sender and the analytics engine already retry across the gap. The backend
repo's `bun run local` does most of this today, so this is packaging. The wrapper is registered as a login task
(scheduled task, launchd, or a systemd user unit), so a reboot brings everything back without a step.

This works only where the prod signing keys are, which is the user's PC. Keys are never copied to a VPS, so a VPS
never uses a quick tunnel; it has a static address and gets a static name through sslip.io or the user's domain.

The explorer on a quick tunnel keeps the admin-token login on (Roblox sign-in needs a fixed public URL). The wizard
says the tunnel is public and relies on the 32-character token and the lockout after five failed logins. Anyone
who wants a fixed URL on a home PC can set up a named Cloudflare tunnel by hand; the docs describe it, the installer
never asks for a Cloudflare token.

**sslip.io.** A public DNS service that answers `backend.1-2-3-4.sslip.io` with `1.2.3.4`. No account, nothing to
create. Caddy issues the certificate. To verify before M3: Let's Encrypt rate limits for sslip.io as a shared
registered domain, and ZeroSSL as Caddy's fallback issuer if they bite.

Cloudflare limits that touch the quick tunnel: 100 MB request bodies and a 125 s proxy read timeout. Ingest bodies
are capped at 2 MB by the backend; the explorer's live streams must send keepalives under 125 s.

### After every path

The two 32-character keys are generated on this PC, written to the game's `.env`, and reach the machine only inside
the env file over SSH or in the local env file. Then `backend setup --url https://<hostname>` runs. The check is
`typetorch servers` after the user joins the game. An optional last question takes a Discord or Slack webhook URL for
alerts.

**Explorer exposure.** With a tunnel the admin UI is public on that hostname. The wizard offers
`TYPETORCH_TOKEN_LOGIN=off` with Roblox sign-in on (the backend default is on), or sets the admin allow list to the user's current address,
and explains the lockout after five failed logins.

### Per-run specifics

- **Linux service:** swap, ufw, Bun under `/opt/bun`, the `typetorch` service user, clone to
  `/opt/typetorch-backend`, `bun install` and the explorer build, `/etc/typetorch/backend.env` with mode 640, the
  unit, then Caddy for the domain or the sslip.io name per Q3, then `GET /healthz` through the public URL. One checklist line per step.
  Idempotent: a rerun skips what exists. `--teardown` reverses it.
- **Coolify:** the wizard detects Coolify itself, no question asked: over SSH or locally it looks for
  `/data/coolify`, the `coolify` and `coolify-proxy` containers, and the panel answering on port 8000. Found: skip
  straight to adding the project. Not found: run Coolify's install script, print the panel address and wait for the
  user to set the admin account. Adding the project then has two ways:
  - **Clicked** (default): the wizard prints the exact resource settings (Docker Compose from the backend repo URL,
    compose location `/compose.yaml`, the hostname: the user's domain or the sslip.io name) and the env block with
    the two keys, waits for Enter, then probes `/healthz` and both keys.
  - **Automated** (optional, when the user pastes a Coolify API token from the panel): the wizard creates the
    project, the Docker Compose resource, the domain and the env variables through Coolify's API and starts the
    deploy, then runs the same probes. Skipped silently when no token is given.
  A rerun on a VPS that already has the resource finds it by name and only re-checks it.
- **This PC:** install Bun and cloudflared if missing, clone the backend next to the game, build the explorer,
  write a local env file, register `typetorch backend run` as a login task. No exposure question.

## 5. Agent handoff

The last phase asks two lines: "what is your game?" and "what should the first version do?". The wizard writes
`AGENT_PROMPT.md` into the repo: what init set up, the ids, the branch map, the rules from `docs/agents/AGENTS.md`,
and the two answers. It prints the prompt and, if `claude` is on PATH, offers to run `claude` with it and the
typetorch plugin marketplace added. The agent works locally only, per the playbook, and keeps the "what you need to
do" list current.

## 6. Rules the wizard obeys

- Secrets are masked on input, never printed, never written to `init.json` or `AGENT_PROMPT.md`.
- Every file write is shown first. Every external action (publish, deploy, SSH) has its own y/N.
- Ctrl+C anywhere leaves a consistent state; the next `init` resumes.
- No GitHub Actions, no hosted CI, nothing leaves the user's machine except to Roblox, their VPS and the tunnel.

## 7. Milestones

| Milestone | Scope | Result |
|---|---|---|
| M1 | `tui.ts`, phases 1 to 5 and 7 | Zero to a live hot-swap with no backend and no doc reading |
| M2 | Phase 6: skip, this PC with the quick-tunnel wrapper, Coolify | A backend for test games and for people on Coolify |
| M3 | Phase 6: Linux service over SSH and local, Caddy with a domain or sslip.io, `--teardown` | A full self-hosted backend from one prompt |
| M4 | Phase 8, repair mode, docs rewrite | fresh-setup.md becomes "run init"; AGENTS.md moves init from Planned to Today |

## 8. Known limits

- Studio and the browser cannot be skipped: creating the experience and the Open Cloud key happen there. The wizard points at the exact page and verifies the result.
- The SSH installer owns a server. It must be idempotent and have `--teardown`, or a half-failed run leaves a VPS the
  user does not understand.
- Password SSH works but is the slow path; the wizard steers to a key file.
- A quick tunnel's explorer URL changes on every start, so Roblox sign-in is off there and bookmarks do not work;
  `typetorch backend run` prints the current address.

## 9. Files

In `cli/src`: `tui.ts`, `commands/init.ts` (the phases), `init/` with one file per phase
(`preflight.ts`, `project.ts`, `roblox.ts`, `keys.ts`, `kernel.ts`, `backend.ts`, `deploy.ts`, `agent.ts`),
`init/ssh.ts` (the session, script upload), `init/state.ts` (`.typetorch/init.json`), and
`commands/backend.ts` gains `backend run` (the backend plus quick tunnel wrapper). Backend repo: `server/` already has `Caddyfile`, `backend.env.example` and `typetorch-backend.service`. A single
idempotent `install.sh` for the Linux service steps (the wizard would upload it) and `cloudflared.service` notes are
planned, not built. Docs: fresh-setup.md and fleet-and-alerts.md point at
init first.
