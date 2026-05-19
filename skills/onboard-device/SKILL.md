---
name: onboard-device
description: Onboard a new device into the user's common-config ecosystem, or re-sync an existing one. Clones the repo if missing, symlinks shared skills + project memory, composes ~/.claude/CLAUDE.md from the shared fragment and the per-device fragment. Idempotent — safe to re-run after a git pull. Use when the user invokes this skill, says "onboard this machine", "sync common-config here", or "set up this device".
---

This skill sets up (or re-syncs) a device against the user's `common-config` repo. It is idempotent — re-running it after a `git pull` refreshes everything in place.

## Before you start

Confirm with the user:
1. **Repo location.** Default: `~/common-config` (= `C:\Users\<USER>\common-config` on Windows, `~/common-config` on Linux/macOS). If they already have it elsewhere, use that path.
2. **Is this a first-time onboard or a re-sync?** First-time = no device folder yet. Re-sync = device folder exists in the repo.

If the repo isn't cloned yet, ask the user for their repo URL and clone it (the user owns the private repo; this skill doesn't know the URL).

## Step 1 — determine the device ID

The device ID picks which `<repo>/devices/<id>/` folder this install reads.

**Default:** lowercased `hostname` (e.g. `mylaptop`).

**Override when:** multiple Claude installs share one physical machine — e.g. a machine has a Windows Claude AND a separate Claude install in WSL. Both return the same `hostname`, but they're distinct environments with different `~/.claude/` trees. Ask the user for an explicit device ID like `mylaptop-wsl`.

Ask the user at the start: "Use default device ID `<device-id>`, or specify an override?" Capture the chosen value as `<device-id>` for the rest of this skill. The device folder is `<repo>/devices/<device-id>/`.

**If `<repo>/devices/<device-id>/` does NOT exist** (first-time onboard):
- Create `<repo>/devices/<device-id>/memory/` (empty dir)
- Create `<repo>/devices/<device-id>/CLAUDE.md`:
  - If `~/.claude/CLAUDE.md` already exists with non-trivial content, ask the user whether to seed the new device CLAUDE.md from it (probably yes) or start blank.
  - Otherwise start with a one-line stub: `# <device-id> — device-specific Claude Code instructions`
- Commit and push these new files at the end (step 7).

## Step 2 — symlink skills

For each subdirectory in `<repo>/skills/` *except* `onboard-device` itself:

**Target link:** `~/.claude/skills/<skill-name>` → `<repo>/skills/<skill-name>`

- If the target path already exists and is a symlink pointing to the correct location: skip.
- If it exists and is a symlink to somewhere else: remove and recreate.
- If it exists and is a real directory/file: back it up to `~/.claude/skills/<skill-name>.backup-<timestamp>` and ask the user before overwriting.

Also symlink `onboard-device` itself — once the bootstrap link exists, the user can re-run this skill on every device.

**Windows:** `New-Item -ItemType SymbolicLink -Path <link> -Target <target>`. Requires Developer Mode enabled OR running PowerShell as admin. Check with `Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock' -Name AllowDevelopmentWithoutDevLicense` (value `1` = on). If off and not admin: stop and tell the user to enable Developer Mode (Settings → System → For developers) or run from an admin shell, then re-run the skill.

**macOS/Linux:** `ln -sfn <target> <link>` (the `-n` matters for re-linking existing symlinked dirs).

### Adopt unowned local skills

After processing every skill in `<repo>/skills/`, scan `~/.claude/skills/` for additional folders that exist as real directories (not symlinks) and don't appear in the repo. For each one found, ask the user:

> "I see `<name>` in your `~/.claude/skills/` but it's not in the common-config repo. Want me to adopt it into the repo so it syncs to your other devices?"

If yes:
1. Move the local folder into `<repo>/skills/<name>/`
2. Create a symlink `~/.claude/skills/<name>` → `<repo>/skills/<name>` (the standard pattern)
3. Stage the new files for the commit in step 7

If no: leave the local folder alone. It stays a local-only skill on this device.

Skip folders that look like they were installed by an external manager (EG: a plugin marketplace with its own metadata) — adopting them would orphan them from their maintainer. When in doubt, ask the user rather than guessing.

## Step 3 — symlink shared memory into the home project

Shared memories live in `<repo>/memory/shared/`. They get symlinked into the home-directory project's memory dir.

The home project's memory dir is `~/.claude/projects/<home-slug>/memory/`, where `<home-slug>` is the cwd slug for `~` (Claude Code generates these; on Windows it's typically `C--Users-<USER>`, on Unix typically `-home-<user>`). Find it by listing `~/.claude/projects/` and matching the slug for the home dir, or create the dir if missing.

For each file in `<repo>/memory/shared/`:
- Symlink `~/.claude/projects/<home-slug>/memory/<file>` → `<repo>/memory/shared/<file>`
- Same idempotency rules as step 2 (skip if correct, replace if wrong, back up if conflicting real file).

Also link the device-specific memory files from `<repo>/devices/<device-id>/memory/` into the same project memory dir.

Then update `~/.claude/projects/<home-slug>/memory/MEMORY.md` to reference each linked file. Preserve any existing entries that aren't from the repo — they may be session-scoped memories the user wants kept.

## Step 4 — compose ~/.claude/CLAUDE.md

This file is **generated**, not symlinked, because it's a composition. The composition logic lives in `<repo>/scripts/refresh.{ps1,sh}` so the SessionStart hook (step 5) can reuse it.

1. If `~/.claude/CLAUDE.md` exists and is not a backup, back it up to `~/.claude/CLAUDE.md.backup-<timestamp>` (one-time safety net; only on first run for this device).
2. Run the platform-appropriate refresh script. Pass `-DeviceId`/`--device-id` only if the user specified an override in step 1:
   - **Windows:** `pwsh <repo>/scripts/refresh.ps1 [-DeviceId <device-id>]`
   - **macOS/Linux:** `bash <repo>/scripts/refresh.sh [--device-id <device-id>]`

   It pulls the repo (ignored if already up to date) and writes `~/.claude/CLAUDE.md` from `claude-md/shared.md` + `devices/<device-id>/CLAUDE.md`.

## Step 5 — install the SessionStart hook

So updates pushed from other devices land on this one without manual intervention, install a `SessionStart` hook in `~/.claude/settings.json` that re-runs the refresh script.

Read existing `~/.claude/settings.json` (create if missing). Add to the `hooks.SessionStart` array (preserve any existing entries):

**On Windows, MUST use the `args` exec form** (not a single `command` string). With a string command, Claude Code passes it through a shell — and on Windows with Git Bash installed, that shell is bash, which strips backslashes from paths (`C:\Users\X` becomes `C:UsersX`). The `args` form spawns the executable directly with no shell in the loop, so backslashes survive.

**Windows:**
```json
{
  "type": "command",
  "command": "pwsh",
  "args": [
    "-NoProfile",
    "-ExecutionPolicy", "Bypass",
    "-File", "<repo-abs-with-double-backslashes>\\scripts\\refresh.ps1",
    "-Quiet"
  ],
  "timeout": 30
}
```
Add `"-DeviceId", "<device-id>"` to the args array if the user specified an override in step 1. Use `"powershell"` instead of `"pwsh"` if PS7 isn't installed. The path needs double-escaped backslashes inside JSON (`C:\\Users\\...`).

**macOS/Linux** (string command is fine — no Git Bash backslash issue):
```json
{
  "type": "command",
  "command": "<repo-abs>/scripts/refresh.sh --quiet [--device-id <device-id>]",
  "timeout": 30
}
```

Substitute `<repo-abs>` with the absolute repo path on this device. Include the device-id flag only if the user specified an override in step 1 — otherwise the script falls back to hostname.

**Verify timing.** After installing, test once: from another device, edit `claude-md/shared.md`, push. On this device, start a fresh Claude session and check the new content is visible in CLAUDE.md from message #1. If it isn't, the hook fires too late — flag this to the user; you'll need to accept a one-session lag or find another sequencing trick.

## Step 6 — verify and report

Final checks:
- `ls -la ~/.claude/skills/` — each skill is a symlink pointing into the repo
- `cat ~/.claude/CLAUDE.md | head -20` — composition looks right
- `ls -la ~/.claude/projects/<home-slug>/memory/` — shared + device memories are linked

Report a short summary: hostname/device-id, repo location, # skills linked, # memories linked, CLAUDE.md size. Note any backups created.

## Step 7 — commit any new device files

If step 1 created a new `devices/<device-id>/` folder:
- `git add devices/<device-id>/` in the repo
- Commit: `Onboard <device-id>`
- Push

Confirm before pushing — this is a remote-affecting action.

## Common failure modes

- **Symlink fails on Windows with "A required privilege is not held":** Developer Mode is off and the shell isn't admin. See step 2.
- **Project memory dir doesn't exist:** Claude Code creates it lazily when it first writes a memory. Create it manually (`mkdir -p`) and proceed.
- **Existing `~/.claude/CLAUDE.md` has content not in the repo:** This is the "first onboard" case — that content is device-specific and should move into `devices/<device-id>/CLAUDE.md` before the compose step overwrites it. Always back up first.

## After onboarding

When the user edits a skill or shared memory through Claude on any device, the change lands in the symlinked repo file directly. Remind them to `git commit && push` from `<repo>` so other devices pick it up on next pull. After a pull, the SessionStart hook handles the CLAUDE.md refresh automatically; skill/memory symlinks update through the repo without any extra step.
