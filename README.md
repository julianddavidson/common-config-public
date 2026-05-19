# common-config-public

A public template for setting up a cross-device Claude Code config sync repo of your own.

## What this is

[Claude Code](https://docs.claude.com/en/docs/claude-code/overview) reads customisations from local files in `~/.claude/`: a top-level `CLAUDE.md`, a `skills/` folder of behaviour modules, and per-project memory files. Use it on more than one computer and each machine builds up its own set independently — a rule taught at work doesn't reach your home laptop.

This template repo gives you the bootstrap to build your own private GitHub repo (`common-config`) that holds the canonical version of those files, symlinked into `~/.claude/` on every machine. Edit a skill anywhere, push, and a SessionStart hook on every other machine pulls the change and rebuilds `CLAUDE.md` at the start of the next session.

## How to use it

Open Claude Code on the machine you want to set up first, then paste:

> "Please follow https://github.com/julianddavidson/common-config-public/blob/main/SETUP.md and set me up with my own common-config repo."

Your Claude reads `SETUP.md`, asks you a few questions (your name, GitHub username, etc.), creates a new **private** repo on your GitHub, and wires this machine into it. Total time: about five minutes.

For subsequent devices, the procedure is shorter — three shell commands and one skill invocation. SETUP.md covers it.

## What's in this repo

```
common-config-public/
├── README.md                          <-- this file
├── SETUP.md                           <-- bootstrap procedure for Claude to follow
├── claude-md/shared.md                <-- identity + global behaviour rules (template)
├── scripts/refresh.{ps1,sh}           <-- pull + recompose CLAUDE.md
├── skills/onboard-device/SKILL.md     <-- onboards each device into your repo
└── docs/common-config-deep-dive.md    <-- full system design, sync mechanics, gotchas
```

The deep-dive doc describes one real working deployment as a case study. Worth a skim if you want to know how the pieces fit before you build your own.

## Licence

This template is provided as-is, public domain / CC0 — use it, fork it, modify it, no attribution required.
