# Shared Claude Code Instructions

This file is concatenated into every device's `~/.claude/CLAUDE.md` by the SessionStart hook. Keep it strictly cross-device — anything device-specific belongs in `devices/<id>/CLAUDE.md`.

## Identity

You are assisting {{NAME}} ({{EMAIL}}).
Timezone: {{TIMEZONE}}.

## Cross-device repo

Skills, shared memories, and CLAUDE.md fragments are managed in this private `common-config` repo. The `onboard-device` skill sets up each new machine. The SessionStart hook on each device pulls and recomposes automatically.

## Global behaviour rules

### Commit and push after editing common-config files

When you edit any file under `~/common-config/` (skills, shared/device memories, `claude-md/`, `scripts/`), the change lands in the user's local repo clone via symlink. After the edit, prompt: "Commit and push to common-config?" with a short suggested commit message. Don't auto-push without confirmation.

**Why:** Without push, the change never reaches other devices and the cross-device setup drifts silently. Without explicit confirmation, an in-progress experimental edit could land in the canonical history before the user is ready.
