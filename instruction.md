# Container instructions for Claude (box)

You are running inside the containerized Claude session for this project. The host launched you with `./bin/box`. Read this file fully before doing anything else.

## Where you are
- The current working directory is the project root, bind-mounted from the host. Files you edit here are visible on the host immediately.
- Your `$HOME` lives on the `box-home` named Docker volume. It persists across runs but is **not** shared with the host. Login credentials, shell history, and Claude config live there.
- Available tools: `zsh` + oh-my-zsh, `bash`, `git`, common build essentials, and the Claude Code CLI.

## What to do first (in this order)

1. **Read this file completely.**
2. **Read the README and any agent docs.** `README.md`, `CLAUDE.md`, `AGENTS.md` if they exist. Skim them for: what the project is, how it's run, how it's tested, and any documented dev workflow.
3. **Survey setup needs (don't install yet).** Look for dependency manifests at the root and figure out what the project would need to run — inside the container, never on the host. Common manifests:
   - Node — `package.json` → `npm install` / `pnpm install` / `yarn`
   - Python — `pyproject.toml` + `uv.lock` → `uv sync`; or `requirements.txt` → `pip install -r requirements.txt`; or `poetry.lock` → `poetry install`
   - Rust — `Cargo.toml` → `cargo build`
   - Go — `go.mod` → `go mod download`
   - Ruby — `Gemfile` → `bundle install`
   - Makefile with `setup` / `bootstrap` / `install` target → run that

   Also check the image: are required system tools (`ffmpeg`, `pandoc`, language runtimes, headless browser deps, etc.) installed? If not, plan a `Dockerfile` edit + `docker compose build` for step 4's "System dependencies" section. **Do not actually run any install or rebuild yet** — record findings in `cheatsheet.md`, surface them to the user, and let them decide (step 5).
4. **Write `cheatsheet.md` with this exact structure.** The cheatsheet is the human-facing TL;DR of how to use this project from *outside* the container. It is project-specific (you own it; the host's `/box` template won't overwrite it). Every section is mandatory — write "None" or "Nothing needed" rather than omitting a section. All command examples MUST be prefixed with `./bin/box ` (or the direnv-shortened form) so the user can copy-paste from the host.

   ```
   # <project> cheatsheet

   ## Quickstart — do you need to do anything?
   <Be honest. Examples: "Nothing — `./bin/box uv run pytest` works on first try."
    Or: "First `uv run …` will sync `.venv` into the bind-mounted root (~30s)."
    Or: "Needs an API token in `$FOO_TOKEN` — set it on the host before launching."
    Mention if a rebuild is required first (see System dependencies below).>

   ## Container customizations you may need

   ### Ports
   <Either "None — CLI tool, nothing listens." OR list the port(s) the project binds and the exact
    `ports:` block to add to `compose.yaml` (host:container). Remind that `compose.yaml` is
    template-owned: changing it requires restarting the container (`docker compose down && up`).>

   ### Volumes / data folders
   <Either "None — bind mount `.:/workspace` is sufficient." OR call out where the project writes
    significant data (downloads, caches, DB files, model weights). Recommend either a subfolder
    pattern inside the existing bind mount, or a dedicated additional mount in `compose.yaml`.>

   ### System dependencies (Dockerfile rebuild)
   <Either "None — the bundled image is sufficient." OR list packages the project needs that the
    image doesn't ship. Give exact `Dockerfile` lines to add and remind that the image must be
    rebuilt afterwards (`docker compose build`). The `<NAME>-home` volume persists across rebuilds
    so the user keeps their Claude login and shell history.>

   ## Run / Test / Lint / Format / Build / Clean
   <Whichever sections apply. Bulleted commands prefixed with `./bin/box `, one-line
    description, exact shell. Omit sections that don't apply to this project.>
   ```

   Show the same summary inline in chat. Do not auto-execute. Mention that `./help.sh` (host-side, at the project root) prints `cheatsheet.md` anytime — and with direnv (see "Tip for the human" below) they can just type `help.sh`.
5. **Then ask the user what they want done.** Don't start changing source files, installing deps, or rebuilding the image without a clear ask.

## Tip for the human (on the host)
`bin/box` and `help.sh` both live at the project root on the host. With [direnv](https://direnv.net/) the user can put them on PATH so the `./` prefix isn't needed. Suggest (but don't auto-write) an `.envrc` like:

```
PATH_add bin
PATH_add "$(pwd)"
```

After `direnv allow`, typing `box` launches the container and `help.sh` prints the cheatsheet from anywhere in the project tree. The `/box` template intentionally ships no `.envrc` because direnv is a personal preference — only the user creates it.

## House rules
- The files `Dockerfile`, `compose.yaml`, `.devcontainer/devcontainer.json`, `bin/box`, `help.sh`, and this `instruction.md` are owned by the host's `/box` template. Don't edit them — changes will be overwritten when the template is refreshed. If you need project-specific guidance, put it in `CLAUDE.md`. `cheatsheet.md` is project-specific and IS yours to write/update.
- You are not the host's Claude session. Treat this as a fresh session scoped to this project only.
- Don't install host-level packages or try to `sudo` outside the container. Install runtime deps inside the container if you need them.
