---
name: skill-manager
description: Analyze the current project and manage installed skills - recommend, enable, disable, and resolve conflicts between similar skills to reduce context overhead. Use when asked to "manage skills", "optimize skills", or "skill-manager".
---

# Skill Manager

Analyze the current project's tech stack, workflow, and development stage, then recommend which installed skills to enable or disable. Resolve conflicts between similar skills by guiding the user through comparisons.

## Language Rule

All user-facing output (recommendations, status descriptions, prompts, reason fields in config) MUST use the user's current conversation language. If the user's CLAUDE.md specifies a language preference, follow that. Skill identifiers (e.g. `superpowers:brainstorming`) always stay in English.

## Execution Flow

### Phase 1: Collect Information

Perform ALL of the following in parallel:

1. **Scan installed skills** — Read `~/.claude/plugins/installed_plugins.json`, then for each plugin's `installPath`, read every `skills/*/SKILL.md` frontmatter (name + description). Also check `.claude-plugin/plugin.json` for `commands` entries. Build a full list of `plugin:skill` identifiers with their descriptions.

2. **Analyze project tech stack** — Check for:
   - `package.json` (Node/JS/TS, frameworks like React/Vue/Next.js, test frameworks)
   - `requirements.txt` / `pyproject.toml` / `setup.py` (Python)
   - `go.mod` (Go)
   - `Cargo.toml` (Rust)
   - `*.sln` / `*.csproj` (C#)
   - Other language-specific config files
   - `Dockerfile`, `docker-compose.yml` (containerization)
   - `.github/workflows/` (CI/CD)
   - `fly.toml`, `vercel.json`, `netlify.toml` (deploy targets)

3. **Read project CLAUDE.md** — If it exists, extract workflow preferences, coding standards, and any explicit skill preferences.

4. **Read recent git history** — Run `git log --oneline -20` to understand:
   - What kind of work is being done (features, bugfixes, refactoring)
   - Development stage (early scaffolding vs mature codebase)
   - Active areas of the codebase

### Phase 2: Classify and Group

Group all skills into functional domains:

| Domain | Example Skills |
|--------|---------------|
| Development Workflow | brainstorming, writing-plans, executing-plans, subagent-driven-development |
| Code Quality | review, simplify, code-reviewer, requesting-code-review, receiving-code-review |
| Testing | test-driven-development, verification-before-completion |
| Debugging | systematic-debugging, investigate |
| Frontend/Design | frontend-design, ui-ux-pro-max, design-review, design-shotgun, design-consultation, plan-design-review |
| Deployment/Ops | ship, land-and-deploy, setup-deploy, canary, document-release |
| Security | cso, guard, careful, freeze, unfreeze |
| Browser/QA | browse, gstack, qa, qa-only, connect-chrome, setup-browser-cookies, benchmark |
| Planning Review | plan-ceo-review, plan-eng-review, autoplan, office-hours |
| Tool Integration | codex, claude-api, learn, retro, loop, find-skills |
| Config/Setup | update-config, claude-hud, codex:setup, setup-deploy |
| Git/Version Control | using-git-worktrees, finishing-a-development-branch |

Within each domain, identify **conflict groups** — skills with significant functional overlap.

### Phase 3: Recommend

For each domain, present one of three outcomes:

**Recommended to enable** — Skill is clearly relevant to this project:
```
✅ superpowers:systematic-debugging
   Reason: [why this skill matches the project]
```

**Conflict detected** — Multiple skills overlap, recommend one:
```
⚡ Frontend/Design conflict:
   Recommended: frontend-design:frontend-design
   Reason: [why this is the better fit]
   
   Alternative: ui-ux-pro-max:ui-ux-pro-max
   Focus: [what this one does differently]
   
   Choose [1] frontend-design / [2] ui-ux-pro-max / [3] Enable both?
```

**Recommended to disable** — Skill is not relevant:
```
💤 superpowers:canary
   Reason: [why not needed]
```

Present results **domain by domain**. After each domain, wait for user confirmation or adjustment before moving on.

### Phase 4: Interactive Confirmation

For skills you cannot confidently classify:
- Ask the user directly, one at a time
- Provide context about what the skill does and why you're unsure

For conflict groups:
- Lead with your recommendation and reasoning
- If the user wants more detail, show a side-by-side comparison of the two skills' capabilities

### Phase 5: Write Configuration

After all domains are confirmed, write TWO configuration files:

#### 5a. Plugin-level disabling (`.claude/settings.local.json`)

This is the **primary mechanism** that actually reduces context overhead. For each installed plugin, check if ALL of its skills are in the disabled list. If so, disable the entire plugin via `enabledPlugins`.

1. Read the existing `.claude/settings.local.json` (or create it).
2. For each plugin, count how many of its skills are enabled vs disabled.
3. If a plugin has **zero enabled skills**, set `"pluginKey": false` in `enabledPlugins`.
4. If a plugin has **any enabled skills**, ensure it is set to `true` (or remove the entry to inherit the global default).
5. **NEVER disable `skill-manager@skill-manager`** — the manager itself must always stay enabled.
6. Write the updated settings file, preserving all existing fields (permissions, env, etc.).

Example result in `.claude/settings.local.json`:
```json
{
  "permissions": { "...existing..." },
  "enabledPlugins": {
    "frontend-design@claude-plugins-official": false,
    "ui-ux-pro-max@ui-ux-pro-max-skill": false
  }
}
```

**Important**: The plugin key format in `enabledPlugins` is `pluginName@marketplaceName` (e.g. `superpowers@claude-plugins-official`). Read `~/.claude/plugins/installed_plugins.json` to get the exact keys — they are the top-level keys in the `plugins` object.

#### 5b. Skill-level profile (`.claude/skill-profile.json`)

For plugins that remain enabled but have some skills disabled, write the fine-grained profile:

```json
{
  "version": 1,
  "createdAt": "<ISO timestamp>",
  "updatedAt": "<ISO timestamp>",
  "enabled": ["plugin:skill", "..."],
  "disabled": ["plugin:skill", "..."],
  "disabledPlugins": ["frontend-design@claude-plugins-official", "..."],
  "conflicts": {
    "<domain>": {
      "chosen": "plugin:skill",
      "over": ["plugin:skill", "..."],
      "reason": "<user's language explanation>"
    }
  },
  "projectContext": {
    "techStack": ["tech1", "tech2", "..."],
    "analyzedAt": "<ISO timestamp>"
  }
}
```

The `disabledPlugins` array records which plugins were fully disabled at the `enabledPlugins` level, so `/skill-status` and `/skill-toggle` can track them.

Create the `.claude/` directory if it doesn't exist. Write both files using the Write tool.

#### 5c. Confirm to user

After writing, show a summary:

```
Configuration saved:

Plugin-level (context fully removed):
  ❌ frontend-design@claude-plugins-official  (all 1 skills disabled)
  ❌ ui-ux-pro-max@ui-ux-pro-max-skill       (all 1 skills disabled)

Skill-level (soft disable via hook):
  💤 superpowers:canary, superpowers:design-shotgun, ...

Enabled: X skills | Disabled: Y skills (Z via plugin-level, W via skill-level)
Restart Claude Code for plugin-level changes to take effect.
```

## Updating Existing Profile

If `.claude/skill-profile.json` already exists when `/skill-manager` is invoked:

1. Read the existing profile
2. Identify new unmanaged skills (installed but not in enabled/disabled lists)
3. Identify stale skills (in profile but no longer installed)
4. Present only the **changes needed**, not a full re-analysis
5. Preserve existing user choices unless the user explicitly wants a full re-analysis

Ask: "Detected N new skills and M removed skills. Update incrementally, or run a full re-analysis?"

## Error Handling

- **No installed plugins file**: Report that no plugins are installed.
- **Corrupted profile**: Warn the user, offer to regenerate from scratch.
- **No git repo**: Skip git history analysis, note this in output.
