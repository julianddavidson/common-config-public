# SETUP — Build the user's own common-config repo from this template

> **Note for humans reading this:** this document is for Claude Code. If you're the user, the way to use it is to open Claude Code on the machine you want to set up first, then paste a message asking Claude to follow this file. The rest of this document is Claude's instructions.

---

# Instructions for Claude

Your goal: build a private GitHub repo (`<user>/common-config`) cloned and wired into `~/common-config/` on this machine, populated from the public template at `julianddavidson/common-config-public`. Then run the `onboard-device` skill to finish wiring this machine.

This must be done carefully — you're creating a remote git repo on the user's GitHub account, writing files into their home directory, and modifying `~/.claude/settings.json`. Confirm intent before destructive steps. Be transparent at each phase.

## Step 1 — confirm intent and gather inputs

Tell the user what you're about to do (one sentence), then ask for these inputs in a single batch:

- **Their name** (used in git commits and the identity block in `shared.md`)
- **Their email** (same)
- **Their GitHub username** (used to create the new private repo)
- **Their timezone** (optional, goes in `shared.md`)
- **This device's ID** — default is the lowercased `hostname`. Ask if they want an override (only needed if multiple Claude installs share one physical machine).

Do not proceed until the user confirms.

## Step 2 — check prerequisites

Verify on this machine:

- `gh auth status` returns authenticated (gh CLI installed and logged into the user's GitHub account)
- `git --version` succeeds
- `~/common-config/` does NOT already exist. If it does, stop and ask — you might overwrite something. Repo location is fixed at `~/common-config/` because scripts and the SessionStart hook assume it.
- **On Windows:** Developer Mode is enabled (Settings → System → For developers). Required for symlinks later. If off, stop and ask the user to enable it before continuing.

## Step 3 — clone the template

Clone this public template to a temporary location, copy the working files into `~/common-config/`, drop the template's `.git` folder so the user's repo starts with fresh history.

```
git clone https://github.com/julianddavidson/common-config-public.git /tmp/common-config-public
mkdir -p ~/common-config
cp -r /tmp/common-config-public/claude-md ~/common-config/
cp -r /tmp/common-config-public/scripts ~/common-config/
cp -r /tmp/common-config-public/skills ~/common-config/
cp -r /tmp/common-config-public/docs ~/common-config/
rm -rf /tmp/common-config-public
```

On Windows PowerShell, the equivalent:

```powershell
git clone https://github.com/julianddavidson/common-config-public.git "$env:TEMP\common-config-public"
New-Item -ItemType Directory -Force "$HOME\common-config" | Out-Null
Copy-Item -Recurse "$env:TEMP\common-config-public\claude-md", `
                   "$env:TEMP\common-config-public\scripts", `
                   "$env:TEMP\common-config-public\skills", `
                   "$env:TEMP\common-config-public\docs" `
          -Destination "$HOME\common-config\"
Remove-Item -Recurse -Force "$env:TEMP\common-config-public"
```

Also create the empty folders the onboarder will populate later:

```
mkdir -p ~/common-config/devices ~/common-config/memory/shared
```

## Step 4 — customise shared.md

Edit `~/common-config/claude-md/shared.md`. Replace the placeholders:

- `{{NAME}}` → the user's name (from step 1)
- `{{EMAIL}}` → the user's email
- `{{TIMEZONE}}` → the user's timezone, or remove the line if they didn't provide one

## Step 5 — write a fresh README for the new private repo

The public template's README is for THIS public repo, not for the user's new private one. Replace `~/common-config/README.md` with a minimal one describing the user's setup. Suggested content:

```markdown
# common-config

Private cross-device Claude Code configuration. Built from the template at https://github.com/julianddavidson/common-config-public.

## Onboarding a new device

1. Install Claude Code on the new machine and sign in.
2. **Windows:** enable Developer Mode (Settings → System → For developers).
3. Clone this repo and link the onboarder skill:

   **macOS/Linux:**
   ```
   git clone <repo-url> ~/common-config
   mkdir -p ~/.claude/skills
   ln -sfn ~/common-config/skills/onboard-device ~/.claude/skills/onboard-device
   ```

   **Windows (PowerShell):**
   ```
   git clone <repo-url> "$HOME\common-config"
   New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
   New-Item -ItemType SymbolicLink -Path "$HOME\.claude\skills\onboard-device" -Target "$HOME\common-config\skills\onboard-device"
   ```

4. Start Claude Code and say: **"run the onboard-device skill"**.

See `docs/common-config-deep-dive.md` for full design notes.
```

Substitute `<repo-url>` with the actual SSH or HTTPS URL of the user's repo once it exists (you'll know after step 7).

## Step 6 — init git locally

```
cd ~/common-config
git init -b main
git -c user.email=<email> -c user.name="<name>" add .
git -c user.email=<email> -c user.name="<name>" commit -m "Initial commit from common-config-public template"
```

Substitute the user's actual name and email.

## Step 7 — create the new private repo and push

Use the gh CLI to create the repo on the user's GitHub account and push the initial commit:

```
gh repo create <github-username>/common-config --private --source=. --push --description "Private cross-device Claude Code configuration"
```

If SSH key isn't set up on the user's GitHub account for this machine, gh's HTTPS auth handles the push.

After this step, the repo exists at `https://github.com/<github-username>/common-config` and contains the template scaffolding.

Now go back and update `~/common-config/README.md` to substitute the actual repo URL where you wrote `<repo-url>` placeholders. Commit and push the README update.

## Step 8 — bootstrap symlink for the onboarder

Create `~/.claude/skills/` if missing, then symlink the onboarder skill into it.

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
New-Item -ItemType SymbolicLink -Path "$HOME\.claude\skills\onboard-device" -Target "$HOME\common-config\skills\onboard-device"
```

**macOS/Linux:**
```bash
mkdir -p ~/.claude/skills
ln -sfn ~/common-config/skills/onboard-device ~/.claude/skills/onboard-device
```

## Step 9 — run the onboarder

Tell the user:

> "Setup complete. Now say 'run the onboard-device skill' and I'll wire this machine into the repo (symlink skills, compose CLAUDE.md, install the SessionStart hook)."

Wait for them to invoke the skill explicitly. Don't proceed unilaterally — the onboarder modifies `~/.claude/` (symlinks, backups of existing files) and the user should be aware that it's running.

## Notes for Claude

- **Don't include any content from `julianddavidson/common-config-public` that isn't part of the template** — specifically, the deep-dive doc is included verbatim as a worked-example reference. It mentions specific devices (PORTIA2022, trc-work1) and the maintainer's name. That's fine for reference but won't reflect the user's deployment.
- **Be careful with Windows backslashes** when writing `~/.claude/settings.json` later (the onboarder handles this, but if you ever edit it manually, use the `args` exec form to bypass shell parsing — see the onboarder SKILL.md step 5).
- **Don't auto-push subsequent edits** — there's a behaviour rule in `shared.md` that says to prompt before each push. Honour it from the start.
- **If anything fails halfway through**, stop and report. The user can re-run from the failed step rather than continuing past a broken state.
