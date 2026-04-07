---
name: skill-status
description: Display the current skill profile status for this project - shows enabled, disabled, unmanaged, and stale skills. Use when asked to "show skill status", "list skills", or "skill status".
---

# Skill Status

Display a formatted status panel showing the current project's skill management state.

## Language Rule

All user-facing output MUST use the user's current conversation language. If the user's CLAUDE.md specifies a language preference, follow that. Skill identifiers always stay in English.

## Execution

1. **Check for profile** — Read `.claude/skill-profile.json` in the current project directory. If it doesn't exist, tell the user no profile is configured and suggest running `/skill-manager`.

2. **Scan installed skills** — Read `~/.claude/plugins/installed_plugins.json` and walk each plugin's `installPath` to build the full list of currently installed `plugin:skill` identifiers.

3. **Read project settings** — Read `.claude/settings.local.json` for `enabledPlugins`. Read `pluginLevels` and `symlinks` from the profile.

4. **Check symlinks** — Verify that symlinks in `.claude/skills/` recorded in the profile still exist and point to valid targets. Flag broken symlinks.

5. **Classify and display** using the three-level structure:

```
📋 Skill Profile Status (project: <project-name>)

L3 — Plugin enabled, all skills active:
  ✅ codex@openai-codex
     codex:codex-rescue, codex:setup, codex:codex-cli-runtime

L2 — Plugin disabled, selected skills symlinked:
  📌 superpowers@claude-plugins-official
     Active (via symlink):  brainstorming, writing-plans, systematic-debugging
     Removed from context:  canary, design-shotgun, design-consultation, ...

L1 — Plugin fully disabled:
  ❌ frontend-design@claude-plugins-official  (1 skill removed)
  ❌ ui-ux-pro-max@ui-ux-pro-max-skill       (1 skill removed)

Unmanaged (<count>):
  <plugin>:<skill>              <description snippet>
  ...
  → Run /skill-manager to include these in your profile.

Stale (<count>):
  <plugin>:<skill>              (plugin uninstalled or symlink broken)
  ...
  → Run /skill-manager to clean up.

Summary: X skills active | Y removed from context | Z via symlink
Last analyzed: <date>  Tech stack: <tech1>, <tech2>, ...
```

If there are no unmanaged or stale skills, show a checkmark:
```
Unmanaged (0): ✓ All skills managed
```

## Error Handling

- **Corrupted profile**: Warn and suggest running `/skill-manager` to regenerate.
- **Missing installed_plugins.json**: Report no plugins installed.
