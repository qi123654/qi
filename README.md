# trellis-demo

基于 [Trellis](https://github.com/) 框架初始化的 AI 辅助开发项目模板，已配置 **Claude Code** 和 **Codex** 两种 AI 助手。

> Trellis 是一个统一管理 Cursor / Claude Code / Codex 等多个 AI 编程助手的工作流框架，通过预置的 commands、skills、agents、hooks 让 AI 按规范的"计划 → 执行 → 收尾"三阶段流程协作开发。

---

## 目录结构

```
trellis-demo/
├── AGENTS.md                # AI 助手项目说明（任何 agent 启动时会读）
├── README.md                # 本文档
├── .claude/                 # Claude Code 配置
│   ├── agents/              #   sub-agent 定义（trellis-implement / check / research）
│   ├── commands/trellis/    #   slash 命令（/trellis:continue, /trellis:finish-work）
│   ├── hooks/               #   会话/工具调用钩子（Python）
│   ├── skills/              #   skills（brainstorm / check / break-loop 等）
│   └── settings.json        #   hooks 注册表
├── .codex/                  # Codex 配置
│   ├── agents/              #   sub-agent TOML 定义
│   ├── hooks/               #   钩子脚本
│   ├── config.toml          #   项目级 Codex 配置
│   └── hooks.json           #   钩子声明
├── .agents/                 # 通用 agents 资源（Cursor/Gemini/Copilot/Amp/Kimi 也读）
│   └── skills/              #   与 .claude/skills/ 对应的镜像
└── .trellis/                # Trellis 框架自身
    ├── workflow.md          #   工作流定义（Phase 1/2/3，per-turn breadcrumb 唯一来源）
    ├── config.yaml          #   项目级配置（自动提交、monorepo、hooks 等开关）
    ├── spec/                #   编码规范（按 package/layer 组织）
    ├── tasks/               #   活动任务（PRD、jsonl 上下文、研究材料）
    ├── workspace/<dev>/     #   每位开发者的会话日志（journal-N.md）
    └── scripts/             #   Python CLI（task.py, get_context.py, ...）
```

---

## 快速开始

### 前置要求

| 工具 | 用途 | 检查命令 |
|---|---|---|
| **Python 3** | 运行 `.trellis/scripts/` 与 hooks | `python --version` |
| **Git** | 版本管理 | `git --version` |
| **Claude Code** 或 **Codex** | AI 助手任选其一 | 各自登录命令 |

### 1. 克隆项目

```bash
git clone <你的仓库地址> trellis-demo
cd trellis-demo
```

### 2. 设置开发者身份（每台设备首次都要做）

```bash
python ./.trellis/scripts/init_developer.py <你的名字>
```

这会创建本地 `.trellis/.developer`（被 gitignore，不进仓库）和 `.trellis/workspace/<你的名字>/`。

### 3a. 使用 Claude Code

直接在项目根目录启动 Claude Code。`SessionStart` 钩子会自动加载 trellis 上下文。常用 slash 命令：

| 命令 | 作用 |
|---|---|
| `/trellis:continue` | 继续当前任务 |
| `/trellis:finish-work` | 收尾（归档任务 + 记录会话） |

### 3b. 使用 Codex

需要在 **用户级** `~/.codex/config.toml` 中开启 hooks 并信任本项目：

```toml
[features]
hooks = true                 # Codex 0.129+；旧版用 codex_hooks = true

[projects."C:/绝对/路径/到/trellis-demo"]
trust_level = "trusted"
```

Codex 0.129+ 还要在 TUI 里运行一次 `/hooks` 批准 Trellis 的 UserPromptSubmit hook，否则工作流面包屑不会自动注入。

---

## 工作流（三阶段）

详细定义见 `.trellis/workflow.md`。简版速记：

### Phase 1: Plan — 想清楚再干

```bash
# 1.0 创建任务
python ./.trellis/scripts/task.py create "<任务标题>" --slug <短名>

# 1.1-1.2 用 trellis-brainstorm skill 与用户对话产出 prd.md，按需研究

# 1.3 整理 implement.jsonl / check.jsonl，把相关 spec 与 research 路径列入

# 1.4 启动任务（状态 → in_progress）
python ./.trellis/scripts/task.py start <task-dir>
```

### Phase 2: Execute — 写代码并验证

- 主流程派发 `trellis-implement` sub-agent 写代码
- 然后派发 `trellis-check` sub-agent 跑 lint / typecheck / 测试
- Codex inline 模式（`codex.dispatch_mode=inline`）下主会话直接编辑

### Phase 3: Finish — 沉淀与收尾

1. **3.1** 最终质量校验
2. **3.2** 反复 debug 时加载 `trellis-break-loop` 复盘
3. **3.3** 用 `trellis-update-spec` 把新发现写回 `.trellis/spec/`
4. **3.4** AI 主导 `git commit`（一次性确认批量提交计划）
5. **3.5** 跑 `/trellis:finish-work`：归档任务 + 记录会话日志

---

## 常用脚本速查

```bash
# 任务管理
python ./.trellis/scripts/task.py create "<title>" --slug <name>
python ./.trellis/scripts/task.py start <task-dir>
python ./.trellis/scripts/task.py current --source
python ./.trellis/scripts/task.py finish
python ./.trellis/scripts/task.py archive <task-dir>
python ./.trellis/scripts/task.py list [--mine] [--status <s>]
python ./.trellis/scripts/task.py --help          # 完整命令

# 上下文 / 阶段指引
python ./.trellis/scripts/get_context.py                            # 当前会话上下文
python ./.trellis/scripts/get_context.py --mode packages            # 列出包与 spec 层
python ./.trellis/scripts/get_context.py --mode phase --step 1.1    # 取某阶段详细指引

# 会话记录
python ./.trellis/scripts/add_session.py --title "T" --commit "<hash>" --summary "..."
```

---

## 迁移到其他设备

本仓库内容**完全可移植**，但目标设备需要做下列适配：

### 必做

1. **装 Python 3** — hooks 调用 `python` 命令（trellis 在 Windows 已默认渲染为 `python`，Linux/Mac 上若实际命令是 `python3`，需调整 PATH 或 alias）。
2. **重新初始化开发者身份**（每台设备每个人一次）：
   ```bash
   python ./.trellis/scripts/init_developer.py <你的名字>
   ```
3. **如果用 Codex**：在新设备 `~/.codex/config.toml` 重新配置 `[features].hooks = true` 和 `[projects."<新绝对路径>"] trust_level = "trusted"`，然后 `/hooks` 批准。

### 已通过 .gitignore 排除的本地状态（不需要也不会随 git 同步）

```
.trellis/.developer
.trellis/.current-task
.trellis/.runtime/
.trellis/.session-id
.trellis/.agent-log
.trellis/.plan-log
.trellis/.ralph-state.json
.trellis/.agents/      # 注：与项目根的 .agents/ 不同
**/__pycache__/
```

### 升级 Trellis 模板

```bash
trellis update             # 拉取最新 trellis 模板（覆盖 .claude/.codex/.agents/.trellis 中的模板部分）
```

---

## 关键文件 / 目录在哪

| 我想... | 去看... |
|---|---|
| 编辑工作流定义 | `.trellis/workflow.md` |
| 调整项目级配置（自动提交、monorepo） | `.trellis/config.yaml` |
| 看 / 写编码规范 | `.trellis/spec/<package>/<layer>/` |
| 看历史任务 | `.trellis/tasks/` 与 `.trellis/tasks/archive/` |
| 看自己的会话日志 | `.trellis/workspace/<你的名字>/journal-N.md` |
| 改 Claude Code hooks | `.claude/settings.json` 与 `.claude/hooks/*.py` |
| 改 Codex hooks | `.codex/hooks.json` 与 `.codex/hooks/*.py` |
| 加新的 slash 命令 | `.claude/commands/trellis/*.md` |
| 看 / 改 skills | `.claude/skills/` 或 `.agents/skills/` |

---

## 常见问题

**Q: 为什么不能在 `~` 主目录里 `trellis init`？**
A: 主目录里的 `.claude/`、`.codex/` 是 CLI 工具的实际运行数据（聊天记录、缓存）。trellis 拒绝在 HOME 目录运行以防误覆盖。

**Q: hooks 报 `python: command not found`？**
A: Linux/Mac 通常是 `python3`。可在系统 PATH 上加 `python -> python3` 软链，或编辑 `.claude/settings.json` 与 `.codex/hooks.json` 把 `python` 改成 `python3`。

**Q: Codex 工作流面包屑没注入？**
A: 检查 `~/.codex/config.toml` 是否开了 `[features].hooks = true`、本项目是否被标记为 `trust_level = "trusted"`、并在 Codex TUI 里跑过 `/hooks` 批准 trellis 的 UserPromptSubmit hook。

**Q: 想在新设备初始化为别人的身份？**
A: 编辑 `.trellis/.developer`，把 `name=` 改掉；或直接 `python ./.trellis/scripts/init_developer.py <新名字>`。

---

## 参考

- 工作流权威定义：`.trellis/workflow.md`
- AI 助手入口：`AGENTS.md`
- Trellis 官方：运行 `trellis --help` 查看本机版本，或访问其官方仓库
