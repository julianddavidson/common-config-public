# common-config deep dive

**Script deep dive / system explainer.**

This document covers the design, layout, and sync mechanics of the `common-config` repo - a private GitHub repository that keeps Claude Code's per-user configuration (skills, memories, instructions) in sync across multiple machines. It's intended for anyone who needs to understand, replicate, or maintain a setup like it.

## BACKGROUND

### WHAT CLAUDE CODE IS

Claude Code is a coding assistant that picks up customisations from local files in `~/.claude/`: a top-level `CLAUDE.md` it reads at the start of every session, a `skills/` folder of reusable behaviour modules (each one a `SKILL.md`), and per-project memory files.

### THE PROBLEM

Use it on more than one computer and each machine builds up its own set of `CLAUDE.md` files, skills files and memory files. A rule taught at work doesn't reach the server or your laptops at home, or other devices you might use. Same assistant with different behaviour per device, and it's a manual chore to make them all behave the same way.

What the user wants is for all his computers to share one setup. One set of skills, memories and other configurations. Make a change on one machine, and the others get it automatically.

### SOLUTION

`common-config` is a private GitHub repo holding the canonical version of every one of those files. On each machine, the live `~/.claude/` folder symlinks into a local clone of the repo - so editing a `SKILL.md` anywhere lands in the repo file. A small hook at the start of each Claude session pulls the latest from GitHub and rebuilds `CLAUDE.md` from a shared piece plus a machine-specific piece. No passwords or keys in the repo - just plaintext config.

### OUTCOME

In practice: tweak a skill or rule on whichever machine is in front of you, and the next session on any other machine has the same tweak. A new device is one clone, one symlink, and one Claude Code session running the `onboard-device` skill - five minutes.

## SCOPE

This article covers `common-config` itself - layout, sync mechanics, design decisions, and the onboarding skill. It does NOT cover:

- The actual content of individual skills (each skill's `SKILL.md` documents itself)
- Project-level Claude config (each project's own `.claude/settings.local.json` and `CLAUDE.md`)
- Credential management - deferred; no shared-secret use case has materialised yet

## ARCHITECTURE OVERVIEW

Three layers stacked on each device:

```
GitHub: <user>/common-config (private)                  <-- canonical source of truth
                ↕  git push / git pull
~/common-config/                                        <-- local clone on this device
                ↕  symlinks (+ generated CLAUDE.md)
~/.claude/                                              <-- live Claude Code config
```

The repo is cloned once per device into `~/common-config/`. The `onboard-device` skill wires the local clone into `~/.claude/` using symlinks (for skills and memory files) and a generated composite (for `CLAUDE.md`). A SessionStart hook re-pulls and re-composes on every session start.

## REPO LAYOUT

```
common-config/
├── README.md                       <-- bootstrap one-liner + multi-device notes
├── claude-md/
│   └── shared.md                   <-- identity + global behaviour rules; prepended to every CLAUDE.md
├── devices/
│   ├── device-a/
│   │   ├── CLAUDE.md               <-- device-a-specific instructions (paths, env quirks)
│   │   └── memory/                 <-- device-a-specific memory files
│   └── device-b/
│       ├── CLAUDE.md
│       └── memory/
├── memory/
│   └── shared/                     <-- project memories that apply across devices
├── scripts/
│   ├── refresh.ps1                 <-- Windows: pull + recompose CLAUDE.md
│   └── refresh.sh                  <-- macOS/Linux equivalent
├── skills/
│   ├── <your-skills>/SKILL.md      <-- one folder per skill
│   └── onboard-device/SKILL.md     <-- the bootstrap skill; sets up new devices
└── docs/
    └── common-config-deep-dive.md  <-- this document
```

## ONBOARDING A DEVICE

The full onboarding flow (run once per new Claude Code install):

1. **Clone the repo.** Land it at `~/common-config/` (the default location every script assumes).
2. **Manually symlink the onboarder skill** into `~/.claude/skills/onboard-device`. This is the bootstrap - one symlink, manually created, so Claude can find the skill to run.
3. **Open Claude Code and say "run the onboard-device skill"**. The skill takes it from there.

What the skill does internally:

- Detects the device id (lowercased `hostname`, or an explicit override for multi-Claude-install machines such as a Windows host plus a WSL install)
- Creates `devices/<id>/` in the repo if it's the first onboard
- Symlinks each skill in `<repo>/skills/` into `~/.claude/skills/` (backs up any pre-existing real folders to `.backup-<timestamp>`)
- Symlinks shared and device-local memories into the home project's memory dir
- Runs `scripts/refresh.{ps1,sh}` to compose `~/.claude/CLAUDE.md` from `shared.md` + the device-specific fragment
- Installs the SessionStart hook in `~/.claude/settings.json`

## SYNC FLOW

### THE PUSH SIDE - GETTING EDITS UPSTREAM

Because `~/.claude/skills/<name>/SKILL.md` is a symlink into the repo, editing the file - whether through Claude or manually - modifies the canonical repo copy directly. There's no separate "sync back" step. From there:

1. `cd ~/common-config && git status` shows the change
2. `git add` + `git commit` + `git push` propagates it to GitHub

A behaviour rule in `shared.md` reminds Claude to prompt the user to commit and push whenever it edits a file under `~/common-config/`. Pushes are never automatic - explicit confirmation is required, because an experimental edit shouldn't slip into the canonical history without intent.

### THE PULL SIDE - GETTING EDITS DOWNSTREAM

Each device's `~/.claude/settings.json` has a `SessionStart` hook that runs `scripts/refresh.ps1` (Windows) or `scripts/refresh.sh` (Unix) at the start of every Claude session. The script:

1. Runs `git -C ~/common-config pull --ff-only --quiet`
2. Reads `claude-md/shared.md` + `devices/<id>/CLAUDE.md`
3. Concatenates them and writes the result to `~/.claude/CLAUDE.md` (UTF-8, no BOM)

Skill symlinks and memory symlinks need no refresh - they point at the repo, so once `git pull` updates the underlying file, the next file read sees the new content.

The composite `CLAUDE.md` is the only piece that requires regeneration. The hook handles that automatically.

## WORKED EXAMPLE - AN EDIT FROM device-a REACHES device-b

To prove the chain end-to-end, walk through a real edit. Assume two onboarded devices: a Windows machine (`device-a`) and a laptop (`device-b`).

**On device-a:**

```
1. User says: "add a behaviour rule that you should always prefer ripgrep over grep"
2. Claude edits ~/.claude/CLAUDE.md ... wait, that's the composed file. Wrong target.
   Correct target: ~/common-config/claude-md/shared.md (the source).
3. Claude opens shared.md (via the editable repo path), adds the rule, saves.
4. Per the behaviour rule "commit and push after editing common-config files",
   Claude asks: "Commit and push?"
5. User confirms. Claude runs:
       git -C ~/common-config add claude-md/shared.md
       git -C ~/common-config commit -m "Add ripgrep-over-grep preference"
       git -C ~/common-config push
6. Commit lands on GitHub.
```

**On device-b, sometime later:**

```
7. User opens Claude Code (any directory).
8. Claude Code fires the SessionStart hook BEFORE loading CLAUDE.md into the prompt.
9. Hook runs:
       pwsh -NoProfile -File C:\Users\<USER>\common-config\scripts\refresh.ps1
            -Quiet -DeviceId device-b
10. refresh.ps1 pulls the repo (gets the ripgrep commit), then rewrites
    ~/.claude/CLAUDE.md from the updated shared.md + devices/device-b/CLAUDE.md.
11. Claude Code reads the freshly-composed CLAUDE.md.
12. From the user's first message, Claude is aware of the new rule.
```

This was validated live by pushing a unique marker token into `shared.md` from one device and confirming the other device's next fresh session quoted it from message #1. Hook timing is correct: the hook fires before `CLAUDE.md` is loaded into the session prompt.

## DESIGN DECISIONS

### SYMLINKS, NOT COPIES

Two reasons. First, edits made anywhere (in `~/.claude/` or in `~/common-config/`) land in the repo file automatically - there's no "sync back" step to forget. Second, after a `git pull`, the new file content is immediately visible through the symlink without any extra step. Copies would double the number of places an edit can go wrong.

The downside is one-time-per-device: Windows needs Developer Mode enabled (Settings -> System -> For developers) so non-admin shells can create symlinks. Worth the tradeoff.

### CLAUDE.MD IS COMPOSED, NOT SYMLINKED

A symlink can only point at one source file, but `~/.claude/CLAUDE.md` needs both `shared.md` (cross-device) AND `devices/<id>/CLAUDE.md` (device-specific) merged together. So `refresh.{ps1,sh}` reads both and writes a composite. The cost is that any edit to either source requires a regenerate step - which is exactly what the SessionStart hook automates.

### SHARED BEHAVIOUR RULES GO INLINE IN CLAUDE.MD, NOT INTO MEMORY

Claude Code's auto-memory system loads memory files per-project (per cwd). So a "shared memory" only loads when you happen to be in the project it's symlinked into. For genuinely global behaviour rules ("commit and push after editing common-config files", "don't re-ask granted Bash permissions"), that's frustrating - they should apply everywhere.

The fix: put global behaviour rules into `claude-md/shared.md` directly. They become part of every device's composed `CLAUDE.md`, which is loaded in every session regardless of cwd. The memory system stays for project-specific facts that genuinely vary by project.

### CREDENTIAL MANAGEMENT DEFERRED

The original plan included a Bitwarden MCP server for syncing Claude-used credentials. On inspection, no actual use case existed - Claude Code itself OAuths per-device, and no project currently in scope needs a shared secret on every machine. Building credential infrastructure ahead of need was overengineering. If a real shared-secret use case appears later, Bitwarden can be added then.

### MULTIPLE CLAUDE INSTALLS PER MACHINE

A single physical machine can host more than one Claude install - EG: a Windows-side install AND a separate Claude install in a WSL distro on the same host. Both report the same `hostname` but have different `~/.claude/` trees. The onboarder accepts an explicit `--device-id` override (EG: `<host>-wsl`) so each Claude install gets its own `devices/<id>/` folder, its own composed `CLAUDE.md`, and its own SessionStart hook entry.

## WINDOWS-SPECIFIC GOTCHAS

These are all real, hit during initial deployment; documented here so the next device avoids them.

### DEVELOPER MODE MUST BE ON

Without it, `New-Item -ItemType SymbolicLink` fails for non-admin users. The onboarder checks at the start of step 2 and stops with a clear message if Developer Mode is off.

### SETTINGS.JSON HOOK PATHS MUST USE THE `args` EXEC FORM

If the SessionStart hook command is a single string like `"pwsh -File C:\\Users\\<USER>\\..."`, Claude Code passes it through a shell. On Windows with Git Bash installed, that shell is bash, which strips backslashes from unquoted paths (`C:\Users\<USER>` becomes `C:Users<USER>`). The hook then can't find the script.

Fix: use the `args` exec form, which spawns the executable directly with no shell in the loop:

```
"hooks": [
  {
    "type": "command",
    "command": "pwsh",
    "args": [
      "-NoProfile",
      "-ExecutionPolicy", "Bypass",
      "-File", "C:\\Users\\<USER>\\common-config\\scripts\\refresh.ps1",
      "-Quiet",
      "-DeviceId", "<device-id>"
    ],
    "timeout": 30
  }
]
```

### UTF-8 ENCODING IN POWERSHELL 5.1

Windows PowerShell 5.1's `Get-Content -Raw` defaults to Windows-1252, which corrupts UTF-8 multi-byte characters (EG: em-dash `—` becomes mojibake `â€"`). `Set-Content -Encoding utf8` writes a UTF-8 BOM that some tools mishandle. `refresh.ps1` therefore uses `Get-Content -Encoding utf8` for reads and `[System.IO.File]::WriteAllText` with `UTF8Encoding($false)` for writes (UTF-8 without BOM).

### PATH GOTCHA WITH HYPHENS

`gh repo clone <name> $HOME\common-config` parses `-config` as a parameter to gh, not part of the path. Quote it: `"$HOME\common-config"`.

## WHAT'S NOT SYNCED

Deliberate exclusions:

- **Credentials and secrets.** Never committed. If shared credentials become necessary later, they go through a vault, not the repo.
- **Project repos.** Each project stays its own GitHub repo, not part of `common-config`.
- **Device-specific config that varies by environment.** Hostname-tied paths, network share credentials, distro names - all live in `devices/<id>/`, not in shared files.
- **Anthropic / Claude Code OAuth tokens.** Each device authenticates separately.

## OPERATIONAL NOTES

- **Repo URL:** `<user>/common-config` on GitHub, private
- **Auth:** SSH on hosts that already have keys configured; HTTPS + Git Credential Manager on new devices that don't
- **Clone location convention:** `~/common-config/` on every device. Don't deviate - scripts and the SessionStart hook assume it
- **Hook script paths:** hardcoded in each device's `~/.claude/settings.json` - if the clone location changes, update the hook entry too
- **Backups:** the onboarder creates `*.backup-<timestamp>` copies of any pre-existing real files it replaces with symlinks. Safe to delete after spot-checking that the repo versions match

## ENHANCEMENTS

Speculative - not implemented:

- **Cross-device message-passing.** A `handoffs/` folder where one device's Claude writes a note for another's Claude to read at next SessionStart. Lets the assistants on different machines coordinate asynchronously without spending API-billed background calls.
- **Vault integration** if a real shared-secret use case appears.
- **A `Stop` hook** that warns at end of session if there are uncommitted or unpushed changes in `~/common-config/` - catches the "forgot to push" failure mode.
