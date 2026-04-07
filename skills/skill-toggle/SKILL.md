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
5. **Symlink handling**: Check the plugin's current level in `pluginLevels`:
   - If **L1** (plugin fully disabled, no skills needed until now): Change to **L2**. Create a symlink for this skill in `.claude/skills/`. Keep `enabledPlugins: false`.
   - If **L2** (plugin already has some symlinks): Create a symlink for this skill. If ALL skills are now enabled, upgrade to **L3**: remove all symlinks, remove `enabledPlugins: false` entry.
   - If **L3**: No symlink needed, skill is already loaded via the plugin.
6. Update `symlinks` in the profile to reflect the new symlink.
7. Update `updatedAt` timestamp.
8. Write the updated profile (and settings if level changed).
9. Confirm with appropriate message. If level changed, note that restart is needed.

**Creating a symlink** (cross-platform):
- macOS / Linux / WSL: `ln -sfn <installPath>/skills/<skill-name> .claude/skills/<skill-name>`
- Windows native: `mklink /J .claude\skills\<skill-name> <installPath>\skills\<skill-name>` (junction, no admin needed)

### Disable

`/skill-toggle disable <plugin:skill>`

1. Read `.claude/skill-profile.json`. If it doesn't exist, tell the user to run `/skill-manager` first.
2. Move the skill from `enabled` to `disabled` in the profile.
3. If the skill was not in either list, add it to `disabled`.
4. **Symlink handling**: Check the plugin's current level:
   - If **L3**: Change to **L2**. Disable plugin via `enabledPlugins: false`. Create symlinks for all REMAINING enabled skills from this plugin. The disabled skill gets no symlink.
   - If **L2**: Remove this skill's symlink from `.claude/skills/`. If no symlinks remain, downgrade to **L1**.
   - If **L1**: No action needed.
5. Update `symlinks` in the profile.
6. Update `updatedAt` timestamp.
7. Write the updated profile and settings.
8. Confirm with appropriate message.

**Removing a symlink** (cross-platform):
- macOS / Linux / WSL: `rm -f .claude/skills/<skill-name>` (only if `test -L` confirms it's a symlink)
- Windows native: `rmdir .claude\skills\<skill-name>` (junctions are removed with rmdir, not del)
- Never delete a real directory. Always verify it is a symlink/junction first.

### Swap

`/skill-toggle swap <current-skill> <replacement-skill>`

1. Read `.claude/skill-profile.json`.
2. Verify both skills exist in installed plugins.
3. Move `<current-skill>` from `enabled` to `disabled`.
4. Move `<replacement-skill>` from `disabled` to `enabled`.
5. **Symlink cascade**: Apply the enable/disable symlink logic for both skills. If they are from different plugins, each plugin's level may change independently. If from the same L2 plugin, remove the old symlink and create the new one.
6. Update the `conflicts` section: set `chosen` to `<replacement-skill>`, add `<current-skill>` to `over`.
7. Update `updatedAt` timestamp.
8. Write the updated profile and settings.
9. Confirm: `🔄 Swapped: <replacement-skill> (enabled) ↔ <current-skill> (disabled).` (Add restart note if any plugin-level changes occurred.)

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
