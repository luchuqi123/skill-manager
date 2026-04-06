# Skill Manager 插件设计文档

## 概述

Skill Manager 是一个 Claude Code 插件，用于管理用户已安装的 skill。解决的核心问题：用户安装了大量 skill，但对当前项目实际有用的只有几个，多余的 skill 会占用上下文窗口。

插件通过智能分析项目特征，推荐适合的 skill 子集，对功能重叠的 skill 引导用户选择，最终生成项目级配置文件，在后续会话中只加载必要的 skill。

## 工作模式

混合模式：
- **自动触发**：SessionStart hook 轻量检测，输出简短提示
- **手动触发**：用户通过 `/skill-manager` 进入深度分析和交互推荐

## 插件结构

```
skill-manager/
├── .claude-plugin/
│   ├── plugin.json              # 插件元数据
│   └── marketplace.json         # 市场注册（后续开源用）
├── skills/
│   ├── skill-manager/           # 主 skill：深度分析 + 交互推荐
│   │   └── SKILL.md
│   ├── skill-status/            # 子命令：查看当前启用状态
│   │   └── SKILL.md
│   └── skill-toggle/            # 子命令：add/remove 单个 skill
│       └── SKILL.md
├── hooks/
│   ├── hooks.json               # hook 配置
│   └── session-start            # 轻量检测脚本
├── package.json
└── README.md
```

## 配置文件

位置：`<项目根>/.claude/skill-profile.json`

```json
{
  "version": 1,
  "createdAt": "2026-04-07T...",
  "updatedAt": "2026-04-07T...",
  "enabled": [
    "superpowers:brainstorming",
    "superpowers:writing-plans",
    "superpowers:test-driven-development"
  ],
  "disabled": [
    "ui-ux-pro-max:ui-ux-pro-max",
    "superpowers:design-shotgun"
  ],
  "conflicts": {
    "frontend-design": {
      "chosen": "frontend-design:frontend-design",
      "over": ["ui-ux-pro-max:ui-ux-pro-max"],
      "reason": "项目使用 Tailwind + React，frontend-design 更匹配"
    }
  },
  "projectContext": {
    "techStack": ["typescript", "react", "tailwind"],
    "analyzedAt": "2026-04-07T..."
  }
}
```

## SessionStart Hook

### 脚本逻辑

1. 读取 `~/.claude/plugins/installed_plugins.json`，统计已安装 skill 总数
2. 检查当前项目目录是否存在 `.claude/skill-profile.json`
3. 根据情况输出不同提示

### 提示场景

**场景 A — 无配置文件 + skill 数量 > 10**：
```
⚡ 检测到 32 个已安装 skill，当前项目尚未配置 skill 优化方案。
   运行 /skill-manager 分析项目并优化 skill 加载。
```

**场景 B — 有配置文件，正常工作**：
```
✓ Skill 优化已启用：12/32 个 skill 活跃
```

**场景 C — 有配置文件但存在未管理的新 skill**：
```
⚡ 检测到 3 个新安装的 skill 尚未纳入管理。
   运行 /skill-manager 更新配置。
```

### 配置生效机制

hook 检测到配置文件存在时，读取 disabled 列表，输出指令文本：
```
[skill-manager] 以下 skill 已被当前项目禁用，请勿使用：
superpowers:design-shotgun, superpowers:canary, ui-ux-pro-max:ui-ux-pro-max, ...
```

此文本作为 hook 输出注入会话上下文，Claude 遵循指令不使用被禁用的 skill。

### hooks.json

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

## 主 Skill：`/skill-manager`

### 触发

用户运行 `/skill-manager`

### 执行流程

#### 1. 收集信息阶段

- 扫描所有已安装 skill（读取各插件的 SKILL.md frontmatter + description）
- 分析项目技术栈（package.json、语言文件、框架配置）
- 读取项目 CLAUDE.md（理解项目特定工作流偏好）
- 读取 git log 最近 20 条提交（理解开发阶段和活跃模式）

#### 2. 智能分类阶段

将所有 skill 按功能域分组：
- 开发流程类（TDD、planning、brainstorming...）
- 代码质量类（review、simplify、debugging...）
- 前端/设计类（frontend-design、ui-ux-pro-max、design-review...）
- 部署/运维类（ship、land-and-deploy、canary...）
- 安全类（cso、guard、careful...）
- 工具/辅助类（browse、gstack、codex...）

识别同类竞争 skill（功能重叠度高的）。

#### 3. 推荐阶段

每个功能域输出推荐结果：
- **推荐启用** — 与项目强相关的 skill + 理由
- **存在冲突** — 展示推荐选择 + 理由，用户可展开对比
- **建议禁用** — 与项目无关的 skill + 理由

用户逐组确认或调整。

#### 4. 交互确认阶段

- 对无法自动判断的 skill，逐一询问用户
- 对冲突组，先给推荐，用户要求时展开详细对比

#### 5. 写入配置

生成 `.claude/skill-profile.json`。

### 冲突对比交互

当用户想了解冲突 skill 的详细差异时：

```
⚡ 前端设计类存在冲突：

  推荐: frontend-design:frontend-design
  理由: 项目使用 React + Tailwind，该 skill 专注于生产级前端代码生成

  备选: ui-ux-pro-max:ui-ux-pro-max
  侧重: 更偏设计系统理论，覆盖 50+ 风格和 161 色板，适合设计探索阶段

  选择 [1] frontend-design / [2] ui-ux-pro-max / [3] 两者都启用？
```

## 子命令 Skill：`/skill-status`

读取 `.claude/skill-profile.json`，输出格式化状态面板：

```
📋 Skill 管理状态（项目: skill-manager）

已启用 (12):
  superpowers:brainstorming          开发流程
  superpowers:writing-plans          开发流程
  superpowers:test-driven-development 开发流程
  superpowers:systematic-debugging   代码质量
  superpowers:ship                   部署运维
  frontend-design:frontend-design    前端设计
  codex:codex-rescue                 工具辅助
  ...

已禁用 (20):
  ui-ux-pro-max:ui-ux-pro-max       前端设计 (冲突: 选择了 frontend-design)
  superpowers:design-shotgun         前端设计
  superpowers:canary                 部署运维
  ...

未管理 (0):  ✓ 所有 skill 已纳入管理

上次分析: 2026-04-07  技术栈: typescript, react, tailwind
```

## 子命令 Skill：`/skill-toggle`

支持增量管理：

- `/skill-toggle enable superpowers:canary` — 启用指定 skill
- `/skill-toggle disable superpowers:ship` — 禁用指定 skill
- `/skill-toggle swap frontend-design ui-ux-pro-max` — 在冲突组中切换选择

直接更新 `skill-profile.json`，无需重新完整分析。

## 错误处理

### 需要处理的场景

1. **项目无 `.claude` 目录** — 首次运行 `/skill-manager` 时自动创建
2. **配置文件损坏/格式错误** — 提示用户，可选重新生成
3. **插件被卸载但配置中仍引用** — `/skill-status` 标记为"已失效"，建议清理
4. **新插件安装后** — hook 检测到未管理的 skill，提示用户更新
5. **无已安装插件** — 提示无需管理

### 不处理的场景（YAGNI）

- 不做 skill 版本追踪
- 不做跨项目配置同步
- 不做自动禁用/启用（始终需要用户确认）

## 组件职责总结

| 组件 | 职责 |
|------|------|
| SessionStart hook | 轻量检测，输出提示/注入禁用指令 |
| `/skill-manager` | 深度分析 + 交互推荐，生成配置 |
| `/skill-status` | 查看当前启用状态面板 |
| `/skill-toggle` | 增量启用/禁用/切换 |
| `.claude/skill-profile.json` | 项目级配置存储 |
