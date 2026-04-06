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

## How It Works

```
Session Start (hook)              Manual Commands
       │                                │
       ▼                                ▼
  Detect profile ──────────►  /skill-manager (analyze)
  Inject directives            /skill-status (view)
       │                       /skill-toggle (edit)
       ▼                                │
  Claude knows which                    ▼
  skills are disabled           .claude/skill-profile.json
```

1. **SessionStart hook** — A lightweight script runs at every session start. It checks for a skill profile and injects disable directives into the session context.
2. **Analysis** — `/skill-manager` performs deep project analysis and walks you through interactive recommendations.
3. **Configuration** — Results are saved to `.claude/skill-profile.json` in your project root.
4. **Enforcement** — On subsequent sessions, the hook tells Claude which skills are disabled for this project.

## Configuration

The profile is stored at `<project-root>/.claude/skill-profile.json`:

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
    "ui-ux-pro-max:ui-ux-pro-max",
    "superpowers:design-shotgun"
  ],
  "conflicts": {
    "Frontend/Design": {
      "chosen": "frontend-design:frontend-design",
      "over": ["ui-ux-pro-max:ui-ux-pro-max"],
      "reason": "Project uses React + Tailwind"
    }
  },
  "projectContext": {
    "techStack": ["typescript", "react", "tailwind"],
    "analyzedAt": "2026-04-07T12:00:00.000Z"
  }
}
```

You can commit this file to your repo so teammates share the same skill profile.

## Multilingual

All user-facing output (recommendations, status panel, conflict descriptions) follows the user's language automatically. If your CLAUDE.md specifies a language preference, Skill Manager respects it. Skill identifiers (e.g. `superpowers:brainstorming`) always remain in English.

## Requirements

- Claude Code v1.0.80+
- Python 3 (for the SessionStart hook script)

## License

MIT
