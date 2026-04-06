---
name: skill-toggle
description: Quickly enable, disable, or swap individual skills without full re-analysis. Use when asked to "enable skill", "disable skill", "swap skill", or "skill-toggle".
---

# Skill Toggle

Incrementally manage individual skills in the project's skill profile without running a full analysis.

## Language Rule

All user-facing output MUST use the user's current conversation language. If the user's CLAUDE.md specifies a language preference, follow that. Skill identifiers always stay in English.

## Commands

This skill is invoked as `/skill-toggle <action> <args>`.

### Enable

`/skill-toggle enable <plugin:skill>`

1. Read `.claude/skill-profile.json`. If it doesn't exist, tell the user to run `/skill-manager` first.
2. Verify the skill identifier exists in the installed plugins (check `~/.claude/plugins/installed_plugins.json` and walk the install path).
3. Move the skill from `disabled` to `enabled` in the profile.
4. If the skill was not in either list (unmanaged), add it to `enabled`.
5. Update `updatedAt` timestamp.
6. Write the updated profile.
7. Confirm: `✅ Enabled <plugin:skill>. Takes effect next session.`

### Disable

`/skill-toggle disable <plugin:skill>`

1. Read `.claude/skill-profile.json`. If it doesn't exist, tell the user to run `/skill-manager` first.
2. Move the skill from `enabled` to `disabled` in the profile.
3. If the skill was not in either list, add it to `disabled`.
4. Update `updatedAt` timestamp.
5. Write the updated profile.
6. Confirm: `💤 Disabled <plugin:skill>. Takes effect next session.`

### Swap

`/skill-toggle swap <current-skill> <replacement-skill>`

1. Read `.claude/skill-profile.json`.
2. Verify both skills exist in installed plugins.
3. Move `<current-skill>` from `enabled` to `disabled`.
4. Move `<replacement-skill>` from `disabled` to `enabled`.
5. Update the `conflicts` section: set `chosen` to `<replacement-skill>`, add `<current-skill>` to `over`.
6. Update `updatedAt` timestamp.
7. Write the updated profile.
8. Confirm: `🔄 Swapped: <replacement-skill> (enabled) ↔ <current-skill> (disabled). Takes effect next session.`

## Error Handling

- **No profile**: Tell user to run `/skill-manager` first.
- **Skill not found**: List similar skill names and ask if user meant one of those.
- **Skill already in target state**: Inform user, no changes needed.

## No Arguments

If invoked as just `/skill-toggle` without arguments, show usage:

```
Usage:
  /skill-toggle enable <plugin:skill>    — Enable a disabled skill
  /skill-toggle disable <plugin:skill>   — Disable an enabled skill
  /skill-toggle swap <skill-a> <skill-b> — Swap one skill for another

Example:
  /skill-toggle enable superpowers:canary
  /skill-toggle disable superpowers:ship
  /skill-toggle swap frontend-design:frontend-design ui-ux-pro-max:ui-ux-pro-max
```
