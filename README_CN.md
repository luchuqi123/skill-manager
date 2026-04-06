# Skill Manager

一个 Claude Code 插件，按项目管理已安装的 skill。分析项目技术栈，推荐相关 skill，解决相似 skill 之间的冲突，禁用不需要的 skill —— 减少不必要的上下文开销。

[English](README.md)

## 解决什么问题

你安装了大量插件和 skill，但对任何一个具体项目来说，真正有用的只有其中几个。每次会话，Claude 都会把所有 skill 加载到上下文中 —— 在你永远不会用到的 skill 上浪费 token。

Skill Manager 通过创建项目级的 skill 配置文件来解决这个问题，告诉 Claude 哪些 skill 该用，哪些该忽略。

## 安装

在 Claude Code 中运行以下命令：

**第一步：添加市场**

```
/plugin marketplace add milan/skill-manager
```

**第二步：安装插件**

```
/plugin install skill-manager@skill-manager
```

**第三步：重新加载插件**

```
/reload-plugins
```

完成！下次会话启动时，插件会自动检测已安装的 skill 数量，并建议运行 `/skill-manager` 进行优化。

## 使用方法

### `/skill-manager` — 分析与推荐

对当前项目运行完整分析，生成 skill 配置：

```
/skill-manager
```

插件会依次执行：

1. **扫描** 所有已安装插件中的 skill
2. **分析** 项目特征（package.json、go.mod、Cargo.toml、CLAUDE.md、git 历史等）
3. **分组** 按功能域归类（开发流程、前端设计、部署运维、安全等）
4. **识别冲突** 找出功能重叠的 skill（如 `frontend-design` vs `ui-ux-pro-max`）
5. **逐域推荐** 建议启用或禁用，附带理由
6. **写入配置** 保存到 `.claude/skill-profile.json`

遇到冲突组时，由你来选择保留哪个：

```
⚡ 前端设计类存在冲突：

  推荐: frontend-design:frontend-design
  理由: 项目使用 React + Tailwind，该 skill 专注于生产级前端代码生成

  备选: ui-ux-pro-max:ui-ux-pro-max
  侧重: 更偏设计系统理论，覆盖 50+ 风格和 161 色板

  选择 [1] frontend-design / [2] ui-ux-pro-max / [3] 两者都启用？
```

如果已有配置文件，`/skill-manager` 会检测新增或已移除的 skill，提供增量更新。

### `/skill-status` — 查看当前状态

```
/skill-status
```

输出格式化的状态面板：

```
📋 Skill 管理状态（项目: my-app）

已启用 (12):
  superpowers:brainstorming          开发流程
  superpowers:writing-plans          开发流程
  superpowers:systematic-debugging   代码质量
  frontend-design:frontend-design    前端设计
  ...

已禁用 (20):
  ui-ux-pro-max:ui-ux-pro-max       前端设计 (冲突: 选择了 frontend-design)
  superpowers:design-shotgun         前端设计
  superpowers:canary                 部署运维
  ...

未管理 (0): ✓ 所有 skill 已纳入管理

上次分析: 2026-04-07  技术栈: typescript, react, tailwind
```

### `/skill-toggle` — 快速切换

无需重新分析，直接管理单个 skill：

```
/skill-toggle enable superpowers:canary
/skill-toggle disable superpowers:ship
/skill-toggle swap frontend-design:frontend-design ui-ux-pro-max:ui-ux-pro-max
```

## 工作原理

```
会话启动 (hook)                  手动命令
       │                            │
       ▼                            ▼
  检测配置文件 ──────────►  /skill-manager (分析)
  注入禁用指令              /skill-status  (查看)
       │                   /skill-toggle  (编辑)
       ▼                            │
  Claude 知道哪些                    ▼
  skill 被禁用             .claude/skill-profile.json
```

1. **SessionStart hook** — 每次会话启动时运行轻量脚本，检测 skill 配置并注入禁用指令到会话上下文。
2. **分析** — `/skill-manager` 对项目进行深度分析，引导你完成交互式推荐。
3. **配置** — 结果保存到项目根目录的 `.claude/skill-profile.json`。
4. **生效** — 后续会话中，hook 会告诉 Claude 哪些 skill 在当前项目中被禁用。

## 配置文件

配置保存在 `<项目根目录>/.claude/skill-profile.json`：

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
    "前端设计": {
      "chosen": "frontend-design:frontend-design",
      "over": ["ui-ux-pro-max:ui-ux-pro-max"],
      "reason": "项目使用 React + Tailwind"
    }
  },
  "projectContext": {
    "techStack": ["typescript", "react", "tailwind"],
    "analyzedAt": "2026-04-07T12:00:00.000Z"
  }
}
```

你可以把这个文件提交到仓库，让团队成员共享相同的 skill 配置。

## 多语言支持

所有面向用户的输出（推荐理由、状态面板、冲突描述等）会自动跟随用户的对话语言。如果你的 CLAUDE.md 中指定了语言偏好，Skill Manager 会遵循该设置。Skill 标识符（如 `superpowers:brainstorming`）始终保持英文原名。

## 系统要求

- Claude Code v1.0.80+
- Python 3（SessionStart hook 脚本需要）

## 许可证

MIT
