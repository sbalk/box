---
description: Set up or retrofit a project with the bundled ghost-box scaffolding (Claude in Docker)
argument-hint: [optional command name — defaults to folder basename]
---

Apply the bundled box template to the current project so it gets a containerized Claude (and zsh shell) with one command. The template is self-contained at `~/.claude/templates/box/` — do NOT read from any ghost-box repo at runtime, and do NOT invoke the `docker-tool-env` skill (it's stale).

Optional argument from the user (may be empty): $ARGUMENTS — if non-empty, use it as `<NAME>` (the wrapper / compose service / volume name). Otherwise default to the project folder's basename.

## 1. Locate the project root

Convention: a folder whose parent is `Projects` (typically `~/Documents/Projects/<name>`). Walk up from `$PWD`:

```bash
d="$PWD"; while [ "$d" != "/" ] && [ "$(basename "$(dirname "$d")")" != "Projects" ]; do d="$(dirname "$d")"; done; [ "$d" = "/" ] && echo "NO_PROJECT_ROOT" || echo "$d"
```

- `NO_PROJECT_ROOT` → stop, ask the user which folder to set up.
- Path equals `.../Projects` itself → stop, ask which subfolder.
- Otherwise that's `<PROJECT_ROOT>`. Set `<NAME>` = `$ARGUMENTS` if provided, else `basename <PROJECT_ROOT>`. Validate `<NAME>` matches `^[a-z][a-z0-9-]*$` (lowercase, digits, dashes; starts with a letter); if it doesn't, stop and ask for a valid name.

All file operations happen inside `<PROJECT_ROOT>` from here on.

## 2. Verify the template exists

```bash
ls ~/.claude/templates/box/Dockerfile ~/.claude/templates/box/compose.yaml ~/.claude/templates/box/.devcontainer/devcontainer.json ~/.claude/templates/box/bin/ghost-box ~/.claude/templates/box/instruction.md ~/.claude/templates/box/help.sh
```

All six must be present. If any are missing, stop and tell the user the template is corrupted at `~/.claude/templates/box/` — don't try to fetch from anywhere else.

The template uses `ghost-box` as the placeholder token throughout (service name, volume name, wrapper filename, references inside files). Step 5 sed-replaces this token with `<NAME>`.

## 2a. Do not read the project's source files

`/box` is project-content-agnostic. While running this command you MUST NOT read, `cat`, `grep`, or otherwise inspect any file inside the project, beyond what's listed below. Do not summarize, analyze, or comment on the project's code, README, or configuration. The user will get a chance to do that *inside* the container, where the in-container Claude reads `instruction.md` and takes it from there.

The only project-side reads `/box` is allowed to perform:

- `ls -A` of the project root (step 3) to detect fresh vs retrofit mode.
- `cmp -s` (byte compare) on the scaffolding files `/box` itself owns: `Dockerfile`, `compose.yaml`, `.devcontainer/devcontainer.json`, `bin/<NAME>`, `instruction.md`, and `help.sh`. This is only to decide create-vs-skip-vs-overwrite.

Nothing else. No `Read` tool calls into the project, no `find`/`grep` over project source, no peeking at `package.json`/`pyproject.toml`/etc. to "tailor" the template — the template is generic on purpose.

## 3. Detect mode

```bash
cd <PROJECT_ROOT> && ls -A
```

- **Empty** (or only `.DS_Store` / similar) → **fresh mode**: render template into the project, `git init`, single initial commit.
- **Non-empty** → **retrofit mode**: render template into a staging dir, then apply only the files that actually differ from what's already in the project. Never touch `.git/`, source code, README, package.json, pyproject.toml, etc.

Report which scaffolding files already exist in `<PROJECT_ROOT>` (`Dockerfile`, `compose.yaml`, `.devcontainer/`, `bin/`, `instruction.md`, `help.sh`). These are the only candidates for change — step 5 will diff them and skip any that are already up to date. Only stat / list these paths; do not read their contents beyond the `cmp -s` allowed in step 2a.

## 4. Render the template into a staging dir

Same in both modes: copy the template into a temp dir, apply the placeholder substitution there, then rename the wrapper. The staging dir is what step 5 compares against the project.

```bash
STAGE="$(mktemp -d)"
mkdir -p "$STAGE/.devcontainer" "$STAGE/bin"
cp ~/.claude/templates/box/Dockerfile                       "$STAGE/Dockerfile"
cp ~/.claude/templates/box/compose.yaml                     "$STAGE/compose.yaml"
cp ~/.claude/templates/box/.devcontainer/devcontainer.json  "$STAGE/.devcontainer/devcontainer.json"
cp ~/.claude/templates/box/bin/ghost-box                    "$STAGE/bin/ghost-box"
cp ~/.claude/templates/box/instruction.md                   "$STAGE/instruction.md"
cp ~/.claude/templates/box/help.sh                          "$STAGE/help.sh"
chmod +x "$STAGE/bin/ghost-box" "$STAGE/help.sh"

sed_inplace() { if sed --version >/dev/null 2>&1; then sed -i "$@"; else sed -i '' "$@"; fi; }
for f in Dockerfile compose.yaml .devcontainer/devcontainer.json bin/ghost-box instruction.md help.sh; do
  sed_inplace "s/ghost-box/<NAME>/g" "$STAGE/$f"
done
if [ "<NAME>" != "ghost-box" ]; then
  mv "$STAGE/bin/ghost-box" "$STAGE/bin/<NAME>"
fi
```

The `sed_inplace` helper handles GNU vs BSD sed (macOS).

## 5. Apply only the files that actually differ

The goal is functional changes, not blind overwrite. For each rendered file, compare with what's already in the project. Skip if byte-identical, write if different (or missing). Track exactly which files changed so the final report is honest.

```bash
cd <PROJECT_ROOT>
mkdir -p .devcontainer bin
CHANGED=(); UNCHANGED=(); CREATED=()

apply() {
  src="$1"; dst="$2"
  if [ ! -e "$dst" ]; then
    cp "$src" "$dst"
    CREATED+=("$dst")
  elif cmp -s "$src" "$dst"; then
    UNCHANGED+=("$dst")
  else
    cp "$src" "$dst"
    CHANGED+=("$dst")
  fi
}

WRAPPER="bin/<NAME>"
apply "$STAGE/Dockerfile"                      "./Dockerfile"
apply "$STAGE/compose.yaml"                    "./compose.yaml"
apply "$STAGE/.devcontainer/devcontainer.json" "./.devcontainer/devcontainer.json"
apply "$STAGE/$WRAPPER"                        "./$WRAPPER"
apply "$STAGE/instruction.md"                  "./instruction.md"
apply "$STAGE/help.sh"                         "./help.sh"
chmod +x "./$WRAPPER" "./help.sh"

# If retrofitting and renaming (e.g. legacy bin/ghost-box still around), leave the old file alone —
# do not delete it. The user can remove it themselves; deletions are not in scope for /box.

rm -rf "$STAGE"
```

If everything lands in `UNCHANGED`, the project is already in sync with the template — that's a successful no-op, not a failure.

## 6. Repo handling

Branch on mode:

### Fresh mode + no `.git/` exists

```bash
cd <PROJECT_ROOT>
git init -q
git add .
git commit -q -m "Initial commit from box template"
```

No `.gitignore` work — the scaffolding IS the project on the very first commit.

### Retrofit mode + `.git/` already exists

Append the box-owned scaffolding paths to `<PROJECT_ROOT>/.gitignore` so they don't pollute the host project's git history. Use leading-slash anchors so the matches are bound to the project root. Substitute `<NAME>` before writing. The set:

```
/Dockerfile
/compose.yaml
/.devcontainer/
/bin/<NAME>
/instruction.md
/cheatsheet.md
/help.sh
```

`cheatsheet.md` is included even though `/box` doesn't write it directly — the in-container Claude generates it on first launch per `instruction.md`, and it's project-private to that workflow.

Idempotency rules:
- Create `.gitignore` if missing.
- If the file already contains the header line `# Added by /box (~/.claude/commands/box.md) — template-owned scaffolding`, don't add it twice. Otherwise append it (preceded by one blank line if the existing file is non-empty).
- For each path line, append only if a literal `grep -Fxq` doesn't already find it.
- Do NOT remove pre-existing lines, even if they match — the user may have added them on purpose.
- Track which lines were actually added (versus already-present) so the final report is honest.

Do NOT run `git init`, `git add`, or `git commit` in retrofit mode. The user's existing repo state is sacrosanct — `.gitignore` is the only file `/box` may touch outside its scaffolding set, and only here.

### Retrofit mode + no `.git/`

Do nothing — there's no repo to ignore into. Don't auto-create `.git/`.

## 7. Final report

Print to the user:

- Mode used (fresh / retrofit) and the resolved `<NAME>`.
- **Created** (full paths) — files that didn't exist before.
- **Changed** (full paths) — files that existed and had a real diff applied. Suggest `git diff` to review.
- **Unchanged** (full paths) — files already byte-identical to the rendered template. If everything is unchanged, say so explicitly: "Project is already in sync with the template; no functional changes needed."
- In retrofit mode: explicitly list what was NOT touched (`.git/` internals, source files, README, etc.). Note: `.gitignore` IS touched in retrofit mode when `.git/` exists — report the lines added (or "already present, no change").
- Usage cheatsheet:
  ```
  ./bin/<NAME>                       # claude (default)
  ./bin/<NAME> --danger              # claude --dangerously-skip-permissions
  ./bin/<NAME> claude --danger       # same, explicit
  ./bin/<NAME> danger                # back-compat positional form
  ./bin/<NAME> zsh                   # shell (oh-my-zsh ready)
  ./bin/<NAME> bash                  # bash shell
  ./bin/<NAME> <anything>            # passed through to the container
  ./help.sh                          # print cheatsheet.md (project-specific commands)
  ```
- Heads-up: first run builds the image (~1 min, includes oh-my-zsh clone); after that, runs are instant. Claude prompts for login on first launch — credentials persist in the `<NAME>-home` named volume across runs and rebuilds.
- Mention that `instruction.md` was placed at the project root and is what the in-container Claude reads first on launch. On that first run it will generate `cheatsheet.md` (project-specific run/build/test commands) — `./help.sh` prints it. Project-specific guidance belongs in `CLAUDE.md`, not in `instruction.md` (which gets refreshed from the template).
- **direnv tip (do not auto-create).** Suggest — but do NOT write — an `.envrc` like:
  ```
  PATH_add bin
  PATH_add "$(pwd)"
  ```
  After `direnv allow`, the user can type `<NAME>` and `help.sh` directly from anywhere in the project, no `./` prefix. The template ships no `.envrc` because direnv is a personal preference; only the user creates it.

## Anti-patterns (never do)

- Do NOT read from `~/Documents/Projects/ghost-box` or any other repo at runtime. The bundled template at `~/.claude/templates/box/` is the only source.
- Do NOT invoke the `docker-tool-env` skill.
- Do NOT write the Dockerfile, compose.yaml, devcontainer.json, or wrapper from scratch — always render from the bundled template so improvements to the template propagate.
- Do NOT `cp` template files directly over project files without diffing first. The point of step 5 is to apply *functional changes*, not to churn mtimes and confuse the user about what really moved.
- Do NOT report a file as "overwritten" or "changed" if it was actually byte-identical to the rendered template. Use the `UNCHANGED` bucket honestly.
- Do NOT `git init` in retrofit mode.
- Do NOT modify the project's source code, `README.md`, `package.json`, or any non-scaffolding file. The single exception is `.gitignore` in retrofit mode with `.git/` present — append-only, idempotent, as specified in step 6.
- Do NOT read, `cat`, `grep`, `find`, or use the `Read` tool against any file inside the project (see step 2a). The only allowed reads are `cmp -s` on the scaffolding files `/box` itself owns. If you catch yourself wanting to "just peek at the README" to write a better report — don't.

## Updating the template later

When ghost-box gains an improvement worth propagating, the user (or a future session) refreshes the bundled template by copying the relevant files from ghost-box (or any other source) into `~/.claude/templates/box/`. No code changes to this command are needed unless the file layout itself changes.
