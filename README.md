# Skill Manager

A Claude Code plugin that manages installed skills per-project. Analyzes your project's tech stack, recommends relevant skills, resolves conflicts between similar ones, and disables the rest — reducing unnecessary context overhead.

[中文文档](README_CN.md)

## The Problem

You've installed dozens of skills across multiple plugins, but only a handful are relevant to any given project. Every session, Claude loads all of them into context — wasting tokens on skills you'll never use.

Skill Manager fixes this by creating a per-project skill profile that tells Claude which skills to use and which to ignore.

## Install

Inside Claude Code, run the following commands:

**Step 1: Add the marketplace**

```
/plugin marketplace add lucas-lu-ai/skill-manager
```

**Step 2: Install the plugin**

```
/plugin install skill-manager@skill-manager
```

**Step 3: Reload plugins and restart**

```
/reload-plugins
```

Then restart Claude Code for the SessionStart hook to take effect. On your next session start, the plugin will detect your installed skills and suggest running `/skill-manager` if optimization is available.

## Usage

### `/skill-manager` — Analyze & Recommend

Run a full analysis of your project to build a skill profile:

```
/skill-manager
```

The plugin will:

1. **Scan** all installed skills across every plugin
2. **Analyze** your project (package.json, go.mod, Cargo.toml, CLAUDE.md, git history, etc.)
3. **Group** skills by domain (Development Workflow, Frontend/Design, Deployment, Security, etc.)
4. **Identify conflicts** between similar skills (e.g. `frontend-design` vs `ui-ux-pro-max`)
5. **Recommend** which to enable or disable, domain by domain
6. **Write** the profile to `.claude/skill-profile.json`

For conflict groups, you choose which skill to keep:

```
⚡ Frontend/Design conflict:

  Recommended: frontend-design:frontend-design
  Reason: Project uses React + Tailwind, this skill focuses on production-grade code

  Alternative: ui-ux-pro-max:ui-ux-pro-max
  Focus: Design system theory, 50+ styles, 161 color palettes

  Choose [1] frontend-design / [2] ui-ux-pro-max / [3] Enable both?
```

If a profile already exists, `/skill-manager` detects new or removed skills and offers an incremental update.

### `/skill-status` — View Current State

```
/skill-status
```

Displays a formatted status panel:

```
📋 Skill Profile Status (project: my-app)

Enabled (12):
  superpowers:brainstorming          Development Workflow
  superpowers:writing-plans          Development Workflow
  superpowers:systematic-debugging   Code Quality
  frontend-design:frontend-design    Frontend/Design
  ...

Disabled (20):
  ui-ux-pro-max:ui-ux-pro-max       Frontend/Design (conflict: chose frontend-design)
  superpowers:design-shotgun         Frontend/Design
  superpowers:canary                 Deployment/Ops
  ...

Unmanaged (0): ✓ All skills managed

Last analyzed: 2026-04-07  Tech stack: typescript, react, tailwind
```

### `/skill-toggle` — Quick Toggle

Manage individual skills without a full re-analysis:

```
/skill-toggle enable superpowers:canary
/skill-toggle disable superpowers:ship
/skill-toggle swap frontend-design:frontend-design ui-ux-pro-max:ui-ux-pro-max
```

- **enable** — Activate a disabled skill for the current project
- **disable** — Deactivate an enabled skill for the current project
- **swap** — Switch between two conflicting skills in one step. For example, if you previously chose `frontend-design` over `ui-ux-pro-max` during analysis, `swap` lets you reverse that choice without re-running `/skill-manager`

## How It Works

Skill Manager uses a **three-level optimization** to maximize context savings:

| Level | Condition | Action | Context effect |
|-------|-----------|--------|----------------|
| **L1** | All skills in plugin disabled | `enabledPlugins: false` | All descriptions removed |
| **L2** | Some skills needed | `enabledPlugins: false` + symlink needed skills to `.claude/skills/` | **Only needed skills in context** |
| **L3** | All skills needed | Keep plugin enabled | All descriptions in context |

**L2 is the key feature.** Claude Code discovers skills from the project's `.claude/skills/` directory. By disabling a plugin and symlinking only the needed skills into that directory, we achieve true skill-level filtering — only the skills you actually use enter the context.

```
/skill-manager analyzes project
       │
       ├── All skills disabled?  ──► L1: enabledPlugins: false
       │                              (all context removed)
       │
       ├── Some skills needed?   ──► L2: enabledPlugins: false
       │                              + symlink needed skills to .claude/skills/
       │                              (only needed skills in context)
       │
       └── All skills needed?    ──► L3: plugin stays enabled
                                      (full context)
```

### Example

You have `superpowers` (20 skills) but only need 3:

```
# Plugin disabled via enabledPlugins: false
# Only these 3 skills enter the context:
.claude/skills/brainstorming  → ~/.claude/plugins/cache/.../superpowers/.../skills/brainstorming
.claude/skills/writing-plans  → ~/.claude/plugins/cache/.../superpowers/.../skills/writing-plans
.claude/skills/systematic-debugging → ~/.claude/plugins/cache/.../superpowers/.../skills/systematic-debugging
```

**Result**: 17 skill descriptions removed from context. The 3 needed skills work as local skills (invoked as `/brainstorming` instead of `/superpowers:brainstorming`).

## Configuration

Three files are managed by the plugin:

### `.claude/settings.local.json` — Plugin-level control

```json
{
  "enabledPlugins": {
    "superpowers@claude-plugins-official": false,
    "frontend-design@claude-plugins-official": false,
    "ui-ux-pro-max@ui-ux-pro-max-skill": false
  }
}
```

### `.claude/skills/` — Symlinked skills (L2)

```
.claude/skills/
├── brainstorming → ~/.claude/plugins/cache/.../skills/brainstorming
├── writing-plans → ~/.claude/plugins/cache/.../skills/writing-plans
└── systematic-debugging → ~/.claude/plugins/cache/.../skills/systematic-debugging
```

### `.claude/skill-profile.json` — Profile metadata

```json
{
  "version": 2,
  "enabled": ["superpowers:brainstorming", "superpowers:writing-plans"],
  "disabled": ["superpowers:canary", "superpowers:design-shotgun"],
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
  "conflicts": { "...": "..." },
  "projectContext": { "techStack": ["typescript", "react"], "analyzedAt": "..." }
}
```

You can commit `skill-profile.json` to your repo so teammates share the same profile. Symlinks and `settings.local.json` are local only.

## Multilingual

All user-facing output (recommendations, status panel, conflict descriptions) follows the user's language automatically. If your CLAUDE.md specifies a language preference, Skill Manager respects it. Skill identifiers (e.g. `superpowers:brainstorming`) always remain in English.

## Requirements

- Claude Code v1.0.80+
- Python 3 (for the SessionStart hook script)

## License

MIT
