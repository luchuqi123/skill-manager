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

3. **Read project settings** — Read `.claude/settings.local.json` to check which plugins are disabled at the `enabledPlugins` level. Also read the `disabledPlugins` array from the profile.

4. **Classify each skill** into one of five states:
   - **Enabled**: in the profile's `enabled` list and currently installed
   - **Disabled (plugin-level)**: belongs to a plugin disabled via `enabledPlugins` — context fully removed
   - **Disabled (skill-level)**: in the profile's `disabled` list but plugin is still enabled — context present but Claude won't use it
   - **Unmanaged**: currently installed but not in either list
   - **Stale**: in the profile but no longer installed

5. **Output the status panel** in this format:

```
📋 Skill Profile Status (project: <project-name>)

Enabled (<count>):
  <plugin>:<skill>              <domain>
  <plugin>:<skill>              <domain>
  ...

Disabled — plugin-level (<count>):              ← context fully removed
  ❌ frontend-design@claude-plugins-official     (1 skill)
  ❌ ui-ux-pro-max@ui-ux-pro-max-skill          (1 skill)

Disabled — skill-level (<count>):               ← soft disable via hook
  💤 superpowers:canary                          Deployment/Ops
  💤 superpowers:design-shotgun                  Frontend/Design
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
