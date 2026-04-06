# Skill Manager 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个 Claude Code 插件，通过项目级配置管理已安装 skill 的启用/禁用状态，减少不必要的上下文占用。

**Architecture:** 插件由 SessionStart hook（轻量检测 + 禁用指令注入）和三个 skill（主分析、状态查看、增量切换）组成。配置存储在项目目录 `.claude/skill-profile.json`。

**Tech Stack:** Bash（hook 脚本）、Markdown（skill 文件）、JSON（配置文件）

---

## 文件结构

```
skill-manager/
├── .claude-plugin/
│   ├── plugin.json              # 插件元数据
│   └── marketplace.json         # 市场注册信息
├── skills/
│   ├── skill-manager/
│   │   └── SKILL.md             # 主 skill：深度分析 + 交互推荐
│   ├── skill-status/
│   │   └── SKILL.md             # 查看当前启用状态
│   └── skill-toggle/
│       └── SKILL.md             # 增量启用/禁用/切换
├── hooks/
│   ├── hooks.json               # hook 事件配置
│   └── session-start            # 轻量检测脚本
├── package.json
└── README.md
```

---

### Task 1: 插件脚手架（plugin.json + package.json + hooks.json）

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.claude-plugin/marketplace.json`
- Create: `package.json`
- Create: `hooks/hooks.json`

- [ ] **Step 1: 创建 `.claude-plugin/plugin.json`**

```json
{
  "name": "skill-manager",
  "description": "Manage installed skills per-project: analyze, recommend, enable/disable skills to reduce context overhead",
  "version": "0.1.0",
  "author": {
    "name": "milan"
  },
  "license": "MIT",
  "keywords": [
    "skills",
    "management",
    "optimization",
    "context"
  ]
}
```

- [ ] **Step 2: 创建 `.claude-plugin/marketplace.json`**

```json
{
  "name": "skill-manager",
  "description": "Skill management marketplace",
  "owner": {
    "name": "milan"
  },
  "plugins": [
    {
      "name": "skill-manager",
      "description": "Manage installed skills per-project: analyze, recommend, enable/disable skills to reduce context overhead",
      "version": "0.1.0",
      "source": "./"
    }
  ]
}
```

- [ ] **Step 3: 创建 `package.json`**

```json
{
  "name": "skill-manager",
  "version": "0.1.0"
}
```

- [ ] **Step 4: 创建 `hooks/hooks.json`**

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/session-start\"",
            "async": false
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 5: 验证目录结构**

Run: `find . -type f | grep -v '.git/' | sort`

Expected:
```
./.claude-plugin/marketplace.json
./.claude-plugin/plugin.json
./docs/superpowers/plans/2026-04-07-skill-manager.md
./docs/superpowers/specs/2026-04-07-skill-manager-design.md
./hooks/hooks.json
./package.json
```

- [ ] **Step 6: 提交**

```bash
git add .claude-plugin/plugin.json .claude-plugin/marketplace.json package.json hooks/hooks.json
git commit -m "feat: add plugin scaffold (plugin.json, package.json, hooks.json)"
```

---

### Task 2: SessionStart Hook 脚本

**Files:**
- Create: `hooks/session-start`

hook 脚本负责三件事：
1. 统计已安装 skill 数量
2. 检查当前项目是否有 `.claude/skill-profile.json`
3. 根据场景输出不同提示 + 注入禁用指令

- [ ] **Step 1: 创建 `hooks/session-start` 脚本**

```bash
#!/usr/bin/env bash
# SessionStart hook for skill-manager plugin
# Detects skill profile status and injects disable directives

set -euo pipefail

INSTALLED_PLUGINS_FILE="${HOME}/.claude/plugins/installed_plugins.json"
PROJECT_DIR="${PWD}"
PROFILE_FILE="${PROJECT_DIR}/.claude/skill-profile.json"

# --- Helper: count total installed skills ---
# Walks each plugin's install path, counts SKILL.md files + command .md files
count_installed_skills() {
  local total=0

  if [ ! -f "$INSTALLED_PLUGINS_FILE" ]; then
    echo "0"
    return
  fi

  # Extract all installPath values from installed_plugins.json
  local paths
  paths=$(python3 -c "
import json, sys
with open('${INSTALLED_PLUGINS_FILE}') as f:
    data = json.load(f)
for entries in data.get('plugins', {}).values():
    for e in entries:
        print(e.get('installPath', ''))
" 2>/dev/null || echo "")

  while IFS= read -r install_path; do
    [ -z "$install_path" ] && continue
    [ ! -d "$install_path" ] && continue

    # Count SKILL.md files in skills/ subdirectories
    if [ -d "${install_path}/skills" ]; then
      local skill_count
      skill_count=$(find "${install_path}/skills" -name "SKILL.md" -maxdepth 2 2>/dev/null | wc -l | tr -d ' ')
      total=$((total + skill_count))
    fi

    # Count command .md files referenced in plugin.json
    if [ -f "${install_path}/.claude-plugin/plugin.json" ]; then
      local cmd_count
      cmd_count=$(python3 -c "
import json
with open('${install_path}/.claude-plugin/plugin.json') as f:
    data = json.load(f)
cmds = data.get('commands', [])
print(len(cmds))
" 2>/dev/null || echo "0")
      total=$((total + cmd_count))
    fi
  done <<< "$paths"

  echo "$total"
}

# --- Helper: list all managed skill identifiers from profile ---
get_managed_skills() {
  python3 -c "
import json
with open('${PROFILE_FILE}') as f:
    data = json.load(f)
for s in data.get('enabled', []):
    print(s)
for s in data.get('disabled', []):
    print(s)
" 2>/dev/null || echo ""
}

# --- Helper: list all currently installed skill identifiers ---
# Format: plugin_name@marketplace:skill_name -> plugin_name:skill_name
get_all_skill_ids() {
  if [ ! -f "$INSTALLED_PLUGINS_FILE" ]; then
    return
  fi

  python3 -c "
import json, os, re

with open('${INSTALLED_PLUGINS_FILE}') as f:
    data = json.load(f)

for key, entries in data.get('plugins', {}).items():
    # key format: 'plugin_name@marketplace_name'
    plugin_name = key.split('@')[0]

    for entry in entries:
        install_path = entry.get('installPath', '')
        if not install_path or not os.path.isdir(install_path):
            continue

        skills_dir = os.path.join(install_path, 'skills')
        if os.path.isdir(skills_dir):
            for name in os.listdir(skills_dir):
                skill_md = os.path.join(skills_dir, name, 'SKILL.md')
                if os.path.isfile(skill_md):
                    print(f'{plugin_name}:{name}')

        # Also check commands in plugin.json
        pj = os.path.join(install_path, '.claude-plugin', 'plugin.json')
        if os.path.isfile(pj):
            with open(pj) as pf:
                pdata = json.load(pf)
            for cmd_path in pdata.get('commands', []):
                cmd_name = os.path.splitext(os.path.basename(cmd_path))[0]
                print(f'{plugin_name}:{cmd_name}')
" 2>/dev/null || echo ""
}

# --- Helper: escape string for JSON ---
escape_for_json() {
  local s="\$1"
  s="\${s//\\\\/\\\\\\\\}"
  s="\${s//\"/\\\\\"}"
  s="\${s//\$'\\n'/\\\\n}"
  s="\${s//\$'\\r'/\\\\r}"
  s="\${s//\$'\\t'/\\\\t}"
  printf '%s' "\$s"
}

# --- Helper: output JSON for Claude Code hook ---
output_context() {
  local context="\$1"
  local escaped
  escaped=\$(escape_for_json "\$context")

  if [ -n "\${CURSOR_PLUGIN_ROOT:-}" ]; then
    printf '{\\n  "additional_context": "%s"\\n}\\n' "\$escaped"
  elif [ -n "\${CLAUDE_PLUGIN_ROOT:-}" ]; then
    printf '{\\n  "hookSpecificOutput": {\\n    "hookEventName": "SessionStart",\\n    "additionalContext": "%s"\\n  }\\n}\\n' "\$escaped"
  else
    printf '{\\n  "additional_context": "%s"\\n}\\n' "\$escaped"
  fi
}

# --- Main logic ---
total_skills=$(count_installed_skills)
SKILL_THRESHOLD=10

# Scenario A: no profile, many skills installed
if [ ! -f "$PROFILE_FILE" ]; then
  if [ "$total_skills" -gt "$SKILL_THRESHOLD" ]; then
    msg="[skill-manager] ${total_skills} skills installed, no profile configured for this project. Run /skill-manager to analyze and optimize."
    output_context "$msg"
  fi
  exit 0
fi

# Profile exists — read disabled list
disabled_list=$(python3 -c "
import json
with open('${PROFILE_FILE}') as f:
    data = json.load(f)
print(', '.join(data.get('disabled', [])))
" 2>/dev/null || echo "")

enabled_count=$(python3 -c "
import json
with open('${PROFILE_FILE}') as f:
    data = json.load(f)
print(len(data.get('enabled', [])))
" 2>/dev/null || echo "0")

# Check for unmanaged skills (Scenario C)
managed_skills=$(get_managed_skills)
all_skills=$(get_all_skill_ids)
unmanaged_count=0
while IFS= read -r skill_id; do
  [ -z "$skill_id" ] && continue
  if ! echo "$managed_skills" | grep -qxF "$skill_id"; then
    unmanaged_count=$((unmanaged_count + 1))
  fi
done <<< "$all_skills"

# Build context message
context_parts=""

# Scenario C: new unmanaged skills
if [ "$unmanaged_count" -gt 0 ]; then
  context_parts="[skill-manager] ${unmanaged_count} new skill(s) not yet managed. Run /skill-manager to update.\n"
fi

# Scenario B: normal status + inject disable directive
if [ -n "$disabled_list" ]; then
  context_parts="${context_parts}[skill-manager] Profile active: ${enabled_count}/${total_skills} skills enabled.\n[skill-manager] The following skills are DISABLED for this project. Do NOT use or suggest them:\n${disabled_list}"
else
  context_parts="${context_parts}[skill-manager] Profile active: ${enabled_count}/${total_skills} skills enabled."
fi

output_context "$context_parts"

exit 0
```

- [ ] **Step 2: 设置可执行权限**

Run: `chmod +x hooks/session-start`

- [ ] **Step 3: 验证脚本语法**

Run: `bash -n hooks/session-start`
Expected: 无输出（语法正确）

- [ ] **Step 4: 提交**

```bash
git add hooks/session-start
git commit -m "feat: add session-start hook for skill profile detection and directive injection"
```

---

### Task 3: 主 Skill `/skill-manager`

**Files:**
- Create: `skills/skill-manager/SKILL.md`

这是最核心的 skill，指导 Claude 完成深度分析和交互推荐流程。

- [ ] **Step 1: 创建 `skills/skill-manager/SKILL.md`**

````markdown
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

After all domains are confirmed, generate `.claude/skill-profile.json`:

```json
{
  "version": 1,
  "createdAt": "<ISO timestamp>",
  "updatedAt": "<ISO timestamp>",
  "enabled": ["plugin:skill", ...],
  "disabled": ["plugin:skill", ...],
  "conflicts": {
    "<domain>": {
      "chosen": "plugin:skill",
      "over": ["plugin:skill", ...],
      "reason": "<user's language explanation>"
    }
  },
  "projectContext": {
    "techStack": ["tech1", "tech2", ...],
    "analyzedAt": "<ISO timestamp>"
  }
}
```

Create the `.claude/` directory if it doesn't exist. Write the file using the Write tool.

After writing, confirm to the user:
```
Configuration saved to .claude/skill-profile.json
Enabled: X skills | Disabled: Y skills
Changes take effect on next session start.
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
````

- [ ] **Step 2: 验证文件格式**

Run: `head -4 skills/skill-manager/SKILL.md`
Expected:
```
---
name: skill-manager
description: Analyze the current project and manage installed skills...
---
```

- [ ] **Step 3: 提交**

```bash
git add skills/skill-manager/SKILL.md
git commit -m "feat: add main skill-manager skill for project analysis and recommendation"
```

---

### Task 4: 子 Skill `/skill-status`

**Files:**
- Create: `skills/skill-status/SKILL.md`

- [ ] **Step 1: 创建 `skills/skill-status/SKILL.md`**

````markdown
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
````

- [ ] **Step 2: 提交**

```bash
git add skills/skill-status/SKILL.md
git commit -m "feat: add skill-status skill for viewing profile state"
```

---

### Task 5: 子 Skill `/skill-toggle`

**Files:**
- Create: `skills/skill-toggle/SKILL.md`

- [ ] **Step 1: 创建 `skills/skill-toggle/SKILL.md`**

````markdown
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
````

- [ ] **Step 2: 提交**

```bash
git add skills/skill-toggle/SKILL.md
git commit -m "feat: add skill-toggle skill for incremental enable/disable/swap"
```

---

### Task 6: 端到端验证

**Files:**
- None (validation only)

- [ ] **Step 1: 验证完整目录结构**

Run: `find . -type f | grep -v '.git/' | grep -v 'docs/' | sort`

Expected:
```
./.claude-plugin/marketplace.json
./.claude-plugin/plugin.json
./hooks/hooks.json
./hooks/session-start
./package.json
./skills/skill-manager/SKILL.md
./skills/skill-status/SKILL.md
./skills/skill-toggle/SKILL.md
```

- [ ] **Step 2: 验证所有 SKILL.md 都有正确的 frontmatter**

Run: `for f in skills/*/SKILL.md; do echo "=== $f ==="; head -4 "$f"; echo; done`

Expected: 每个文件都有 `---` 开头的 YAML frontmatter，包含 `name` 和 `description`。

- [ ] **Step 3: 验证 hook 脚本可执行**

Run: `ls -la hooks/session-start`

Expected: 权限包含 `x`（如 `-rwxr-xr-x`）。

- [ ] **Step 4: 验证 hook 脚本语法**

Run: `bash -n hooks/session-start`

Expected: 无输出。

- [ ] **Step 5: 验证 JSON 文件格式**

Run: `python3 -c "import json; [json.load(open(f)) for f in ['.claude-plugin/plugin.json', '.claude-plugin/marketplace.json', 'package.json', 'hooks/hooks.json']]" && echo "All JSON valid"`

Expected: `All JSON valid`

- [ ] **Step 6: 最终提交（如有未提交的改动）**

Run: `git status`

如果有未提交的改动，提交它们。
