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

3. **Classify each skill** into one of four states:
   - **Enabled**: in the profile's `enabled` list and currently installed
   - **Disabled**: in the profile's `disabled` list and currently installed
   - **Unmanaged**: currently installed but not in either list
   - **Stale**: in the profile but no longer installed

4. **Output the status panel** in this format:

```
📋 Skill Profile Status (project: <project-name>)

Enabled (<count>):
  <plugin>:<skill>              <domain>
  <plugin>:<skill>              <domain>
  ...

Disabled (<count>):
  <plugin>:<skill>              <domain> (conflict: chose <other-skill>)
  <plugin>:<skill>              <domain>
  ...

Unmanaged (<count>):
  <plugin>:<skill>              <description snippet>
  ...
  → Run /skill-manager to include these in your profile.

Stale (<count>):
  <plugin>:<skill>              (plugin uninstalled)
  ...
  → Run /skill-manager to clean up.

Last analyzed: <date>  Tech stack: <tech1>, <tech2>, ...
```

If there are no unmanaged or stale skills, show a checkmark:
```
Unmanaged (0): ✓ All skills managed
```

## Error Handling

- **Corrupted profile**: Warn and suggest running `/skill-manager` to regenerate.
- **Missing installed_plugins.json**: Report no plugins installed.
