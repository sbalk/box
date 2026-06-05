# ghost-box

A boilerplate claude in docker repo as peace of mind for all the recent supply chain attacks. Assuming docker is safe, otherwise we'll probably find out soon enough at this rate.

It's for running [Claude Code](https://docs.claude.com/en/docs/claude-code) in a project-scoped Docker container. Works as both a plain `docker compose` setup and a VS Code / JetBrains Dev Container.

## Quick start

```bash
git clone <this-repo-url> new-project-name
cd new-project-name
./rename.sh                      # rebrand container/volume/wrapper + reset git history
./bin/new-project-name           # (the wrapper got renamed too)
# OR
./bin/new-project-name danger    # to run with --dangerously-skip-permissions
```

The first time `claude` starts inside the container, it will prompt you to log in. Your credentials live in the project's named volume and persist across runs and rebuilds.

## What this protects against (and what it doesn't)

Most recent attacks — bad npm packages, malicious editor extensions, "paste this command to fix it" tricks — work by running on your machine as you, with full access to your SSH keys, browser logins, cloud credentials, and every other project on disk. ghost-box boxes that reach in: anything nasty runs inside a throwaway container that only sees this one project and its own little config volume.

It does **not** protect you from:

- **The project files themselves.** `/workspace` is the same folder on your disk, so a compromised tool can read, change, or leak anything in it — including secrets in `.env`, local DB dumps, or notes you keep alongside the code.
- **Git push rights.** If `git` inside the box can push to your repos, so can malware inside the box — it can quietly slip a commit into a branch, tag a poisoned release, or open a PR as you.
- **Anything you've logged into from inside the box.** GitHub, npm, PyPI, AWS, GCP, Cloudflare, Vercel, your database — once those credentials are in the container, a bad dependency can publish packages, deploy code, drop tables, or rack up a cloud bill in your name.
- **Production and other systems you can reach.** Deploy scripts, kubectl contexts, SSH keys to staging boxes, webhook URLs — if your project can reach them, so can whatever runs in the container.
- **Stuff that happens on someone else's machine.** Your CI runners, GitHub Actions, build servers, and registries are outside the box entirely; this does nothing for compromises that happen there.
- **Data leaving over the network.** The container has normal internet access. Source code, tokens it scrapes, environment variables — all of it can be sent out.
- **Commands pasted into your normal terminal.** If a "fix this" snippet ends up in your host shell instead of the container shell, you're back to running as yourself.
- **Persistence inside the box.** The config volume sticks around on purpose so your Claude login survives restarts. A backdoor written into shell rc files or `~/.claude/` lives there too until you wipe the volume.
- **Holes in Docker itself.** Container escapes are rare, but they do happen — a kernel bug, a Docker/runc CVE, or a misconfiguration can let something in the box break out onto the host. Unlikely today, almost certainly going to be in the news at some point.

Think of it as a seatbelt, not a vault: a big improvement over running everything as yourself, but not a substitute for paying attention.

## What's in the image

- `@anthropic-ai/claude-code` (the `claude` CLI)
- `node` 24 (base image)
- `python3` + `uv`
- `git`, `ripgrep`, `less`, `curl`, `ca-certificates`
- `make`, `shellcheck`
- `build-essential` (so MCP servers with native deps compile cleanly)

## Prerequisites

- Docker Desktop, OrbStack, Colima, or another Docker-compatible engine
- `docker compose` (bundled with Docker Desktop / OrbStack)

## Usage

### Run Claude (default)

```bash
./bin/ghost-box
```

### Drop into a shell instead

```bash
./bin/ghost-box bash
```

You'll get an interactive shell at `/workspace`. `claude` is on `$PATH` if you want to launch it manually. `exit` leaves the container; the named volume keeps your shell history and `~/.claude` across sessions.

### YOLO mode

```bash
./bin/ghost-box danger
```

Launches Claude with `--dangerously-skip-permissions`, so it stops asking before every tool call. The whole point of running in a box is that this is now a survivable choice — Claude can romp around `/workspace` and the volume freely, and everything under [What this protects against (and what it doesn't)](#what-this-protects-against-and-what-it-doesnt) still applies. Don't use this flag on the host.

### Run a one-off command

```bash
./bin/ghost-box bash -c "uv pip install requests && python3 script.py"
```

### Shorter command (optional)

If typing `./bin/ghost-box` gets old, add a shell alias scoped to your project. For example, in `~/.zshrc`:

```bash
alias gb="$HOME/path/to/new-project-name/bin/ghost-box"
```

Project-scoped helpers like [direnv](https://direnv.net/) (`PATH_add bin`) can put `bin/` on `$PATH` automatically when you `cd` into the project — useful if you work across several boxes.

### Use with VS Code / JetBrains Dev Containers

Open the folder and let the IDE pick up `.devcontainer/devcontainer.json`. The same Dockerfile is reused, and the same `ghost-box-home` volume holds your config.

## Renaming the project

You probably don't want every clone to be called `ghost-box`. There's an optional `rename.sh` that updates all references in one go:

```bash
./rename.sh                # uses the current folder name
./rename.sh my-thing       # or pick an explicit name
```

It updates `Dockerfile`, `compose.yaml`, `.devcontainer/devcontainer.json`, this README, and renames `bin/ghost-box`. It also wipes the template's `.git` directory, runs `git init`, and makes a single initial commit so you start with a clean history. After it runs, also rename the folder itself (it tells you the exact command), then push to GitHub:

```bash
gh repo create my-thing --private --source=. --push    # or do this via the GitHub UI
```

## Notes

- **First build is slow, later runs are instant.** The image is cached; `docker compose run --rm` only creates a fresh container each time.
- **Persistence model.** Anything inside `/workspace` is your project on disk (bind mount). Anything inside `/home/node` — config, history, `~/.claude/` — lives in the `ghost-box-home` named volume. The container itself is ephemeral.
- **macOS / Windows bind-mount performance.** `/workspace` is a bind mount across the Docker Desktop VM boundary. Fine for editing; noticeable for very heavy I/O.
- **No `direnv` required.** This repo doesn't ship a `.envrc` — call the wrapper directly with `./bin/ghost-box`.
- **Running `/box` on this repo itself.** The `/box` skill writes a `.gitignore` that excludes the box scaffolding (`/Dockerfile`, `/compose.yaml`, `/.devcontainer/`, `/bin/<name>`, `/instruction.md`, `/cheatsheet.md`, `/help.sh`) — sensible when retrofitting an external project, but counter-productive here since those files **are** the template source. If you run `/box` on ghost-box, delete `.gitignore` (or remove those lines) before committing, or git won't show changes to the new scaffolding files.
