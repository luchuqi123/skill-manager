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
/plugin marketplace add luchuqi123/skill-manager
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

Skill Manager uses a **two-level disable mechanism** to maximize context savings:

| Level | Mechanism | Effect |
|-------|-----------|--------|
| **Plugin-level** | `.claude/settings.local.json` `enabledPlugins` | Skill descriptions **completely removed** from context |
| **Skill-level** | `.claude/skill-profile.json` + SessionStart hook | Descriptions still in context, but Claude won't use them |

When `/skill-manager` analyzes your project, it automatically determines the optimal level:
- If **all skills** in a plugin are irrelevant → disable the entire plugin (plugin-level)
- If **some skills** in a plugin are needed → keep plugin enabled, disable individual skills (skill-level)

```
/skill-manager analyzes project
       │
       ├── All skills in plugin disabled? ──► enabledPlugins: false
       │                                      (context fully removed)
       │
       └── Some skills still needed? ──► skill-profile.json
                                          (soft disable via hook)
```

### Session lifecycle

1. **Plugin loading** — Claude Code reads `.claude/settings.local.json` and skips disabled plugins entirely. Their skill descriptions never enter the context.
2. **SessionStart hook** — For plugins that remain enabled, the hook injects disable directives for individual skills.
3. **Result** — Claude only sees skills that are actually relevant to your project.

## Configuration

Two files are managed by the plugin:

### `.claude/settings.local.json` — Plugin-level control

```json
{
  "enabledPlugins": {
    "frontend-design@claude-plugins-official": false,
    "ui-ux-pro-max@ui-ux-pro-max-skill": false
  }
}
```

### `.claude/skill-profile.json` — Skill-level control

```json
{
  "version": 1,
  "createdAt": "2026-04-07T12:00:00.000Z",
  "updatedAt": "2026-04-07T12:00:00.000Z",
  "enabled": [
    "superpowers:brainstorming",
    "superpowers:writing-plans",
    "superpowers:systematic-debugging"
  ],
  "disabled": [
    "superpowers:design-shotgun",
    "superpowers:canary"
  ],
  "disabledPlugins": [
    "frontend-design@claude-plugins-official",
    "ui-ux-pro-max@ui-ux-pro-max-skill"
  ],
  "conflicts": {
    "Frontend/Design": {
      "chosen": null,
      "over": ["frontend-design:frontend-design", "ui-ux-pro-max:ui-ux-pro-max"],
      "reason": "Pure backend project, no frontend code"
    }
  },
  "projectContext": {
    "techStack": ["typescript", "node", "express"],
    "analyzedAt": "2026-04-07T12:00:00.000Z"
  }
}
```

You can commit `skill-profile.json` to your repo so teammates share the same profile. Note that `settings.local.json` is typically gitignored (local only).

## Multilingual

All user-facing output (recommendations, status panel, conflict descriptions) follows the user's language automatically. If your CLAUDE.md specifies a language preference, Skill Manager respects it. Skill identifiers (e.g. `superpowers:brainstorming`) always remain in English.

## Requirements

- Claude Code v1.0.80+
- Python 3 (for the SessionStart hook script)

## License

MIT
