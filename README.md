# OpenCode Swarm — 多智能体协作开发环境安装指南

> 一键搭建 architect + coder + designer 三智能体 + CodeGraph 语义索引 + Beads 任务图 + Muninn 持久记忆 的完整 AI 编码工作流。

---

## 架构概览

```
┌────────────────────────────────────────────────────┐
│                    OpenCode CLI                      │
├────────────────────────────────────────────────────┤
│  Architect (primary)                                │
│  ├─ 规划需求、拆解任务、分配到 Coder/Designer         │
│  ├─ 使用 Beads (bd) 管理任务依赖图                   │
│  └─ 使用 Muninn 持久化架构决策                        │
│                                                     │
│  Coder (subagent)                                   │
│  ├─ 实现业务逻辑、测试、重构                          │
│  ├─ 使用 CodeGraph 理解代码库                        │
│  └─ 遵循 TDD 流程                                    │
│                                                     │
│  Designer (subagent)                                │
│  ├─ 产出可访问、响应式的 UI 脚手架                    │
│  └─ 为 Coder 提供 TODO 占位的组件结构                 │
├────────────────────────────────────────────────────┤
│  MCP 集成                                           │
│  ├─ CodeGraph MCP — 语义代码索引                     │
│  ├─ Beads MCP — 图结构任务跟踪                       │
│  └─ Muninn MCP — 跨会话持久记忆                      │
└────────────────────────────────────────────────────┘
```

---

## 第一步：环境要求

| 依赖 | 版本要求 | 说明 |
|------|---------|------|
| **OpenCode** | 最新版 | `npm install -g opencode` |
| **Node.js** | >=18 | CodeGraph 自带运行时，无需 Node 也可安装 |
| **Python + uv** | >=3.10 | 用于 Muninn MCP 和 Beads MCP |
| **Git** | >=2.0 | 项目版本管理 |

```bash
# 安装 opencode
npm install -g opencode

# 安装 uv（Python 包管理器）
curl -LsSf https://astral.sh/uv/install.sh | sh

# 确认所有工具可用
opencode --version
uv --version
```

---

## 第二步：安装 CodeGraph（语义代码索引）

CodeGraph 为项目构建预索引知识图谱，所有智能体通过 MCP 协议调用，避免用 grep/read 遍历代码。

```bash
# 运行交互式安装器（自动检测 opencode）
npx @colbymchenry/codegraph

# 或者手动非交互安装
npm install -g @colbymchenry/codegraph
codegraph install --target=opencode --location=global --yes
```

安装器会：
1. 自动检测你已安装的智能体（包括 opencode）
2. 将 MCP 服务器配置写入 `~/.config/opencode/opencode.json`
3. 将 `codegraph` 加入 PATH

**重启 opencode** 后 MCP 服务器生效。

在每个项目中初始化索引：

```bash
cd /path/to/your-project
codegraph init -i
```

这会在项目根目录创建 `.codegraph/` 并构建初始索引。此后 opencode 智能体会自动使用 CodeGraph 工具。

---

## 第三步：安装 Beads（任务图管理）

Beads（`bd`）= 面向 AI 智能体的分布式图结构任务追踪器。Architect 用它创建、分配、管理任务依赖。

```bash
# 方式一：Homebrew（推荐）
brew install beads

# 方式二：一键脚本
curl -fsSL https://raw.githubusercontent.com/gastownhall/beads/main/scripts/install.sh | bash

# 方式三：npm
npm install -g @beads/bd
```

### 在项目中初始化

```bash
cd /path/to/your-project
bd init
```

### 为 OpenCode 注入指令

```bash
bd setup opencode
```

这会在项目 `AGENTS.md` 中写入 Beads 工作流指令段，Architect 读取后会使用 `bd create`, `bd show`, `bd close` 等命令管理任务。

### 常用命令速查

```bash
bd ready              # 列出无阻塞、待处理的任务
bd create "标题" -t feature -p 1    # 创建任务（-t: bug/feature/task, -p: 0-3）
bd show bd-42 --json  # 查看任务详情
bd update bd-42 --claim -q   # 认领任务
bd close bd-42 --reason "Done" -q  # 关闭任务
bd prime              # 打印当前工作流上下文
```

---

## 第四步：安装 Muninn Memory（跨会话持久记忆）

Muninn 是本地运行的记忆数据库，智能体用它保存架构决策、编码规范、已知坑点，跨会话复用。

> **注意**：Muninn 目前通过 HTTP MCP 连接。你需要自己部署 Muninn 服务（本地 Docker 或源码运行），然后配置到 opencode。

### 4.1 启动 Muninn 服务

```bash
# 使用 Docker（推荐）
docker run -d --name muninn \
  -p 8750:8750 \
  -v muninn_data:/data \
  ghcr.io/your-org/muninn:latest

# 或从源码运行（需要有 Go 环境）
git clone https://github.com/your-org/muninn.git
cd muninn
go run ./cmd/server
```

服务启动后监听 `http://127.0.0.1:8750`。

### 4.2 在 opencode.json 中配置

```json
{
  "mcp": {
    "memory": {
      "type": "remote",
      "url": "http://127.0.0.1:8750/mcp",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer <your-muninn-token>"
      }
    }
  }
}
```

---

## 第五步：配置 opencode 核心环境

### 5.1 目录结构

```
~/.config/opencode/
├── opencode.json          # 全局配置（模型、MCP、智能体）
├── AGENTS.md              # 全局智能体行为规范（CodeGraph 指令等）
├── agents/
│   ├── architect.md       # Primary 智能体 — 编排者
│   ├── coder.md           # Subagent — 代码实现
│   └── designer.md        # Subagent — UI/UX 脚手架
├── commands/              # 自定义 slash 命令
├── skills/                # 技能目录
└── plugins/               # 插件目录
```

### 5.2 全局配置文件 `opencode.json`

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "autoupdate": false,
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "agent": {
    "general": {
      "model": "deepseek/deepseek-v4-pro"
    }
  },
  "mcp": {
    // CodeGraph — 由 codegraph install 自动写入
    "codegraph": {
      "type": "local",
      "command": ["codegraph", "serve", "--mcp"],
      "enabled": true
    },
    // Muninn Memory
    "memory": {
      "type": "remote",
      "url": "http://127.0.0.1:8750/mcp",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer <your-muninn-token>"
      }
    }
    // Beads 通过 CLI 命令调用，无需 MCP 配置
  }
}
```

> **说明**：CodeGraph MCP 配置通常由 `codegraph install --target=opencode` 自动写入，你无需手动添加。Muninn 需要手动配置。

### 5.3 安装智能体定义文件

将三个智能体文件复制到 agents 目录：

```bash
mkdir -p ~/.config/opencode/agents

# 从本仓库复制智能体定义文件
cp agents/architect.md ~/.config/opencode/agents/
cp agents/coder.md ~/.config/opencode/agents/
cp agents/designer.md ~/.config/opencode/agents/
```

**智能体角色说明**：

| 智能体 | 类型 | 职责 |
|--------|------|------|
| **architect** | `primary` | 理解需求 → 探索代码 → 拆解为 bd 任务 → 异步分派给 coder/designer |
| **coder** | `subagent` | 拉取 bd 任务 → 加载 TDD + grill-me 技能 → 实现代码 → 验证 → 提交 |
| **designer** | `subagent` | 拉取 UI 设计任务 → 加载 frontend-design + grill-me 技能 → 产出可访问的响应式脚手架 |

### 5.4 全局行为规范 `AGENTS.md`

`AGENTS.md` 是所有对话的全局前缀。包含：
- CodeGraph 工具使用规范（GREP GATE）
- 工作流状态机（INITIAL → EXPLORE → EDIT → VALIDATE）
- Muninn 记忆协议
- TDD 纪律

复制到你的配置目录：

```bash
cp AGENTS.md ~/.config/opencode/AGENTS.md
```

---

## 第六步：项目级配置

每个需要使用 OpenCode Swarm 的项目都需要一份 `.opencode/opencode.json`。

### 最简单的项目配置

在项目根目录创建 `.opencode/opencode.json`：

```jsonc
{
  "$schema": "https://opencode.ai/config.json"
}
```

这个空配置告诉 OpenCode 该项目已激活。全局配置（智能体、MCP）会自然继承。

### 可选：项目级覆盖

你可以在项目配置中覆盖模型、MCP 设置等：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-4-6",
  "mcp": {
    "codegraph": {
      "type": "local",
      "command": ["codegraph", "serve", "--mcp"],
      "enabled": true
    }
  }
}
```

### 项目级行为规范

创建 `.opencode/AGENTS.md`（可选，会在对话中注入）：

```bash
# 例如写入 Beads 的操作指令
echo "Use 'bd' for task tracking. Run 'bd prime' for workflow context." >> .opencode/AGENTS.md
```

---

## 第七步：初始化你的项目

假设你的项目在 `/path/to/my-project`：

```bash
cd /path/to/my-project

# 1. 初始化 CodeGraph 语义索引
codegraph init -i

# 2. 初始化 Beads 任务追踪
bd init
bd setup opencode

# 3. 创建 .opencode 项目配置
mkdir -p .opencode
printf '{\n  "$schema": "https://opencode.ai/config.json"\n}\n' > .opencode/opencode.json

# 4. （可选）写入项目级行为规范
echo '## Issue Tracking
Use `bd` for task tracking: `bd ready` for unblocked work, `bd create`, `bd close`.
Run `bd prime` at session start for workflow context.

## CodeGraph
This project has CodeGraph initialized (.codegraph/ exists).
Use `lsp_codegraph_explore` as your PRIMARY code exploration tool.' >> .opencode/AGENTS.md
```

---

## 第八步：启动并使用

```bash
cd /path/to/my-project
opencode
```

进入 opencode 交互界面后，Architect 作为 primary 智能体接管对话。你可以这样开始：

```
帮我实现用户登录功能，包含邮箱+密码登录和 Google OAuth 登录
```

Architect 会：
1. 调用 `memory_muninn_recall` 检查之前的架构决策
2. 使用 CodeGraph 探索代码库
3. 用 `bd create` 将需求拆解为多个任务
4. 逐个异步分派给 coder 和 designer

---

## 附录 A：技能（Skills）说明

智能体启动时自动加载技能，改变其工作模式：

| 技能 | 加载者 | 效果 |
|------|--------|------|
| `tdd` | coder | 强制遵守红-绿-重构循环，无测试 = 未完成 |
| `grill-me` | coder, designer | 允许智能体向 architect 发起追问，不瞎猜 |
| `frontend-design` | designer | 注入设计规范清单、无障碍检查、响应式策略 |
| `caveman` | (按需) | 超级压缩对话，省 ~75% token |
| `diagnose` | (按需) | 系统化调试：复现→缩小→假设→打桩→修复→回归 |
| `vue` | (按需) | Vue 3 Composition API 专项 |
| `quasar-skilld` | (按需) | Quasar UI 框架专项 |

---

## 附录 B：完整目录参考

```
~/.config/opencode/
├── opencode.json           # 全局配置
├── AGENTS.md               # 全局智能体行为规范
├── agents/
│   ├── architect.md        # 编排智能体，primary
│   ├── coder.md            # 代码实现智能体，subagent
│   └── designer.md         # UI 设计智能体，subagent
├── commands/
│   ├── caveman.md          # /caveman 命令
│   ├── caveman-commit.md   # /caveman-commit 命令
│   └── caveman-review.md   # /caveman-review 命令
├── skills/                 # 技能文件
├── plugins/                # 插件文件
└── opencode-swarm.md       # 本文档

项目目录/
├── .opencode/
│   └── opencode.json       # 项目配置
├── .opencode/AGENTS.md     # （可选）项目级行为规范
├── .codegraph/             # CodeGraph 索引（codegraph init -i 生成）
├── .beads/                 # Beads 数据库（bd init 生成）
└── AGENTS.md               # Beads 工作流指令（bd setup opencode 生成）
```

---

## 附录 C：常见问题

### Q: CodeGraph MCP 提示 "not initialized"
```bash
cd your-project && codegraph init -i
```

### Q: bd 命令找不到
```bash
# Homebrew
brew install beads

# 或检查 PATH
export PATH="$HOME/.local/bin:$PATH"
```

### Q: Muninn 内存连接失败
确保 Muninn 容器/服务在运行：
```bash
docker ps | grep muninn
curl http://127.0.0.1:8750/mcp
```

### Q: Architect 不派发任务给 Coder
- 检查 `agents/architect.md` 中 `mode: primary` 是否设置
- 检查 `agents/coder.md` 中 `mode: subagent` 是否设置
- 确保 opencode 版本支持 multi-agent 模式

### Q: 如何验证整套环境就绪
```bash
# 1. 检查各组件
codegraph status .    # CodeGraph 索引状态
bd ready              # Beads 是否有挂起任务
opencode --version    # OpenCode 版本

# 2. 进入 opencode 对话后
# Architect 应自动：memory_muninn_recall → 读取 AGENTS.md
# 后续应使用 lsp_codegraph_* 工具探索代码
```
