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

After all domains are confirmed, apply the **three-level optimization** and write configuration files.

#### Three-level strategy

For each plugin, determine which level to apply:

| Level | Condition | Action | Context effect |
|-------|-----------|--------|----------------|
| **L1** | ALL skills in plugin are disabled | `enabledPlugins: false` | All descriptions removed |
| **L2** | SOME skills needed, some not | `enabledPlugins: false` + symlink needed skills to `.claude/skills/` | Only needed skills in context |
| **L3** | ALL skills are needed | Keep plugin enabled (no entry in `enabledPlugins`) | All descriptions in context |

**L2 is the key innovation.** By disabling the plugin AND symlinking only the needed skills into the project's `.claude/skills/` directory, Claude Code discovers them as local skills. Their descriptions enter the context, but the rest of the plugin's skills do not.

#### 5a. Create symlinks for L2 plugins (`.claude/skills/`)

For each L2 plugin:

1. Get the plugin's `installPath` from `~/.claude/plugins/installed_plugins.json`.
2. For each **enabled** skill in that plugin, create a directory symlink:

   **macOS / Linux / WSL:**
   ```bash
   ln -sfn <installPath>/skills/<skill-name> .claude/skills/<skill-name>
   ```

   **Windows (native, not WSL):**
   ```cmd
   mklink /D .claude\skills\<skill-name> <installPath>\skills\<skill-name>
   ```
   Note: On Windows, `mklink /D` may require administrator privileges or Developer Mode enabled. If symlinks fail, fall back to **directory junction** (`mklink /J`) which does not require elevated privileges.

   **Cross-platform detection:** Check the OS by reading the platform. If `os.name == 'nt'` or `COMSPEC` is set, use Windows commands. Otherwise use Unix `ln -sfn`.

3. For each **disabled** skill, ensure NO symlink exists in `.claude/skills/` (remove if present).

**Example**: Plugin `superpowers` has 20 skills, you only need `brainstorming` and `writing-plans`:
```
.claude/skills/brainstorming → ~/.claude/plugins/cache/.../superpowers/.../skills/brainstorming
.claude/skills/writing-plans → ~/.claude/plugins/cache/.../superpowers/.../skills/writing-plans
```

**Naming note**: Symlinked skills lose their plugin prefix. `superpowers:brainstorming` becomes just `brainstorming` as a local skill. Inform the user of this in the output.

**Safety rules**:
- Only create symlinks for skills from L2 plugins. Do NOT symlink L3 plugin skills (they're already loaded via the plugin).
- Never overwrite an existing non-symlink directory in `.claude/skills/` — warn the user if a name collision exists.
- If two L2 plugins have skills with the same name, warn the user and ask which to keep.

#### 5b. Update `.claude/settings.local.json`

1. Read the existing file (or create it).
2. Set `enabledPlugins` entries:
   - **L1 plugins**: `"pluginKey": false`
   - **L2 plugins**: `"pluginKey": false` (plugin disabled, skills accessed via symlinks)
   - **L3 plugins**: remove entry from `enabledPlugins` (inherit global default). Do NOT write `true`.
3. **NEVER disable `skill-manager@skill-manager`**.
4. Preserve all existing fields (permissions, env, etc.).

**Important**: The plugin key format is `pluginName@marketplaceName` (e.g. `superpowers@claude-plugins-official`). Read `~/.claude/plugins/installed_plugins.json` for exact keys — they are the top-level keys in the `plugins` object.

#### 5c. Write `.claude/skill-profile.json`

```json
{
  "version": 2,
  "createdAt": "<ISO timestamp>",
  "updatedAt": "<ISO timestamp>",
  "enabled": ["plugin:skill", "..."],
  "disabled": ["plugin:skill", "..."],
  "pluginLevels": {
    "superpowers@claude-plugins-official": "L2",
    "frontend-design@claude-plugins-official": "L1",
    "codex@openai-codex": "L3"
  },
  "symlinks": {
    "brainstorming": {
      "source": "superpowers:brainstorming",
      "target": "~/.claude/plugins/cache/.../skills/brainstorming"
    }
  },
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

- `pluginLevels`: records the optimization level per plugin
- `symlinks`: records which local skills are symlinked from which plugin, so `/skill-status` and `/skill-toggle` can manage them

#### 5d. Confirm to user

```
Configuration saved:

L1 — Plugin fully disabled (context removed):
  ❌ frontend-design@claude-plugins-official  (1 skill)
  ❌ ui-ux-pro-max@ui-ux-pro-max-skill       (1 skill)

L2 — Plugin disabled, needed skills symlinked (context optimized):
  📌 superpowers@claude-plugins-official
     Symlinked: brainstorming, writing-plans, systematic-debugging (3 skills)
     Removed: canary, design-shotgun, ... (17 skills)
     Note: These skills are now invoked as /brainstorming instead of /superpowers:brainstorming

L3 — Plugin fully enabled:
  ✅ codex@openai-codex (all 3 skills)

Enabled: X skills | Context savings: Y skill descriptions removed
Restart Claude Code for changes to take effect.
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
