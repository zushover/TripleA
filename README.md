# Triple A (AutodlAgents)

> **AI Agent GPU 云管理桌面应用** — 自然语言操控 GPU 实例，多 Agent 协作，共享记忆，一键部署 Claude Code。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Python](https://img.shields.io/badge/Python-3.12-blue)
![Vue](https://img.shields.io/badge/Vue-3.5-green)
![Tauri](https://img.shields.io/badge/Tauri-2.0-orange)

---

## 这是什么

Triple A 是一个桌面应用，把 GPU 云管理变成对话。不用手动 SSH、不用记命令行，对着 AI Agent 说话就行。

**核心理念：面板是大脑，服务器 Agent 是手脚。** 电脑端 Master Agent 负责任务理解和分发，服务器端子 Agent 负责执行。两边通过 ChromaDB 共享记忆，通过 MCP 协议通信。

```
你说："检查所有 GPU 利用率，空闲的关机"
  → Master Agent 自主决策：
    1. 调 list_gpu_instances 列出所有实例
    2. 逐个 SSH 采集 GPU 利用率
    3. 识别利用率 <10% 的实例
    4. 询问确认后关机
    5. 汇总报告
```

---

## 架构

```
┌──────────────────────────────────────────────────────────────┐
│                  Triple A 桌面端 (Tauri + Vue 3)              │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │          Master Agent (LangGraph ReAct)                │  │
│  │  8 个 Tool：list/check/execute/delegate/shutdown/...    │  │
│  │  模型：DeepSeek V4 Flash (兼容 Anthropic API)           │  │
│  └────────────────────┬───────────────────────────────────┘  │
│                       │                                      │
│  ┌────────────────────┴───────────────────────────────────┐  │
│  │           Memory Hub (ChromaDB 向量数据库)              │  │
│  │  对话历史 · 实验记录 · 文档索引 · 决策日志               │  │
│  └────────────────────┬───────────────────────────────────┘  │
│                       │                                      │
│  ┌────────────────────┴───────────────────────────────────┐  │
│  │           MCP Server (GPU Monitor)                     │  │
│  │  5 个 Tool + 3 个 Resource + 2 个 Prompt                │  │
│  └────────────────────┬───────────────────────────────────┘  │
│                       │                                      │
│  ┌────────────────────┴───────────────────────────────────┐  │
│  │           Vue 3 面板 (5 个页面)                         │  │
│  │  服务器 · Triple A · 知识库 · 费用 · 设置                │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Python Sidecar：FastAPI + SQLite + Paramiko + LangChain     │
│  桌面壳：Tauri 2 (Rust) + 系统托盘                          │
└──────────────────────┬───────────────────────────────────────┘
                       │ SSH + REST API
┌──────────────────────┴───────────────────────────────────────┐
│                  GPU 服务器 (AutoDL)                         │
│                                                              │
│  ┌─────────────────────────┐  ┌─────────────────────────┐   │
│  │  Server A · 3080Ti      │  │  Server B · 4090D       │   │
│  │                         │  │                         │   │
│  │  Claude Code (tmux)     │  │  Claude Code (tmux)     │   │
│  │  Watchdog (任务轮询)     │  │  Watchdog (任务轮询)     │   │
│  │                         │  │                         │   │
│  │  ☀️ 一键部署             │  │  ☀️ 一键部署             │   │
│  └─────────────────────────┘  └─────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### 通信流

```
用户 → Master Agent → 选 Tool → SSH 到服务器 → nvidia-smi/日志 → Agent 分析 → 回答

委派任务：
Master Agent → delegate_to_server(uuid, 任务)
  → SSH → claude -p "任务" 在远程执行
  → 服务器 CC 本地运行 → 返回结果
  → Master 汇总呈现
```

---

## 功能

### 🖥 服务器管理
- **三源实例注册：** Pro API / Web 控制台 / 自定义 SSH
- **智能探测：** SSH 连接 → GPU 检测 → 系统信息采集 → 自动更新状态
- **GPU 监控：** 实时利用率、显存、温度、进程（nvidia-smi）
- **SSH 命令解析：** 粘贴连接字符串 → 自动提取 host/port/user
- **空闲检测：** GPU <5% → 告警 → 自动关机

### 🤖 Triple A（Master Agent）
- **自然语言管理 GPU：** "找到空闲 GPU 并关机"
- **LangGraph ReAct 循环：** 观察 → 思考 → 行动 → 观察
- **8 个 Tool：** list_gpu_instances、check_gpu_utilization、probe_instance_health、execute_on_server、delegate_to_server、get_balance_and_cost、shutdown_idle_instance、save_to_knowledge_base
- **流式 SSE 输出：** 逐字渲染，实时可见
- **子 Agent 委派：** `delegate_to_server()` 远程调服务器 Claude Code

### ☀️ 一键部署 Claude Code
- SSH → 检测环境 → 安装 Node.js → `npm install -g @anthropic-ai/claude-code`
- SFTP 配置 DeepSeek API（`~/.claude/env.sh`）
- tmux 启动（`claude-code` + `watchdog` 双会话）
- 部署后服务器 CC 自动通过 MCP 回连面板

### 🧠 共享记忆 (ChromaDB)
- **3 个 Collection：** conversations / experiments / agent_decisions
- **REST API：** 任何 Agent 可读写搜索
- **自动存档：** 每轮对话自动持久化
- **循环检测：** 语义相似度检查防死循环

### 📡 MCP Server
- **5 个 Tool：** gpu_status、gpu_history、list_gpu_instances、probe_instance、get_balance
- **3 个 Resource：** instances://list、gpu://{uuid}/latest、balance://overview
- **标准 MCP 协议：** 任何 MCP Client 可发现调用

### 📚 知识库
- 对话/实验/决策/文档 全文搜索
- 分类筛选：全部 / 对话 / 实验 / 决策 / 文档
- Agent 专属记忆（预留）

### 💰 费用追踪
- AutoDL：实时余额 + 累计消费
- DeepSeek：API 余额查询（赠送 + 充值）
- 总计余额合并显示

---

## 技术栈

| 层 | 技术 |
|---|------|
| 桌面壳 | Tauri 2 (Rust) |
| 前端 | Vue 3 + TypeScript + Vite |
| 后端 | Python 3.12 + FastAPI |
| 数据库 | SQLite (WAL 模式) |
| Agent 框架 | LangChain + LangGraph |
| 记忆 | ChromaDB (向量数据库) |
| SSH | Paramiko |
| LLM | DeepSeek V4 Flash (兼容 Anthropic API) |
| MCP | Model Context Protocol (自建 Server) |
| 打包 | PyInstaller (sidecar) + Tauri bundler (NSIS) |

---

## 快速开始

### 环境要求
- Python 3.12+
- Node.js 20+
- Rust（Tauri 打包用，可选）
- AutoDL 账号（或任意可 SSH 的 GPU 服务器）

### 开发模式

```bash
# 1. 克隆
git clone https://github.com/zushover/TripleA.git
cd TripleA

# 2. 安装依赖
pip install -r requirements.txt
npm install

# 3. 配置
cp config.example.yaml config.yaml
# 编辑 config.yaml：
#   auto_dl.token: AutoDL 开发者 Token
#   llm.api_key: DeepSeek API Key (sk-xxx)
#   llm.api_base: https://api.deepseek.com/v1
#   llm.model: deepseek-v4-flash

# 4. 启动后端
python sidecar.py

# 5. 启动前端（另一个终端）
npm run dev

# 6. 打开浏览器
# http://127.0.0.1:8899
```

### 生产构建

```bash
npm run build
pyinstaller sidecar.spec --distpath src-tauri/target/debug
cp -r dist src-tauri/target/debug/
cp src-tauri/target/debug/python-sidecar.exe src-tauri/binaries/
cd src-tauri && cargo build --release
```

> 浏览器测试和 Tauri 打包用的是同一份代码。Tauri WebView 加载 `dist/` 静态文件，Python sidecar 作为子进程运行。

---

## 项目结构

```
TripleA/
├── autodl_manager/          # Python 后端
│   ├── api_server.py        # FastAPI (44 条路由)
│   ├── agent/               # Agent 模块（核心）
│   │   ├── tools.py         # 8 个 LangChain Tool
│   │   ├── agent_loop.py    # LangGraph ReAct 循环
│   │   ├── memory.py        # ChromaDB 记忆层
│   │   ├── mcp_server.py    # MCP GPU Monitor Server
│   │   ├── deploy.py        # ☀️ 一键部署 CC
│   │   ├── watchdog.py      # 服务器任务监听器
│   │   ├── executor.py      # 服务器端执行器
│   │   ├── multi_agent.py   # 多 Agent 编排器
│   │   ├── prompts.py       # System Prompt 模板
│   │   └── observability.py # LangFuse 追踪
│   └── ...
├── src/                     # Vue 3 前端
│   ├── App.vue              # 根组件（状态中枢）
│   └── components/
│       ├── Dashboard.vue    # 服务器管理页
│       ├── AgentLog.vue     # Triple A 对话界面
│       ├── KnowledgeBase.vue# 知识库浏览器
│       ├── CostAnalysis.vue # 费用追踪
│       └── ...
├── src-tauri/               # Tauri Rust 桌面壳
├── sidecar.py               # PyInstaller 入口
├── config.example.yaml      # 配置模板
└── requirements.txt
```

---

## Agent Tool 全集

| Tool | 功能 | 需要 |
|------|------|------|
| `list_gpu_instances` | 列出所有 GPU 实例 | — |
| `check_gpu_utilization(uuid)` | 实时 GPU 利用率/显存/温度 | SSH |
| `probe_instance_health(uuid)` | SSH 探测 + 系统信息 | SSH |
| `execute_on_server(uuid, cmd)` | 远程执行命令 | SSH |
| `delegate_to_server(uuid, task)` | 委派任务给服务器 CC | SSH + CC |
| `get_balance_and_cost` | AutoDL 余额 + 消费 | API Token |
| `shutdown_idle_instance(uuid)` | 关机 | SSH |
| `save_to_knowledge_base(cat, title, content)` | 写入知识库 | — |

---

## 实测验证

所有 Agent 功能经真实硬件测试：
- 实例列表 + GPU 探测 → 实时 3080Ti 数据 ✅
- Agent 查询 → LangGraph ReAct 真实 Tool 调用 ✅
- ☀️ 一键部署 CC → 成功部署到 GPU 服务器 ✅
- 知识库 → ChromaDB 对话/实验存储 ✅
- MCP Server → 标准协议 tools/resources ✅
- 费用 → AutoDL + DeepSeek 双余额 ✅
- 流式输出 → SSE 逐字渲染 ✅

---

## License

MIT

---

# Triple A (AutodlAgents)

> **AI Agent GPU Cloud Management Desktop App** — Natural language GPU control, multi-agent orchestration, shared memory, one-click Claude Code deployment.

A desktop application that turns GPU cloud management into a conversation. Instead of SSH-ing into servers manually, you talk to an AI agent that manages everything.

**Core idea:** Panel is the brain. Servers are the limbs. Master Agent on the desktop orchestrates, Sub-agents on servers execute. ChromaDB provides shared memory. MCP provides standardized communication.

### Quick Start (English)

```bash
git clone https://github.com/zushover/TripleA.git
cd TripleA
pip install -r requirements.txt
npm install
cp config.example.yaml config.yaml
# Edit config.yaml with your AutoDL token and DeepSeek API key
python sidecar.py      # Backend on :8899
npm run dev            # Frontend dev server
# Open http://127.0.0.1:8899
```

### Tech Stack

| Layer | Tech |
|-------|------|
| Desktop | Tauri 2 (Rust) |
| Frontend | Vue 3 + TypeScript + Vite |
| Backend | Python FastAPI + SQLite |
| Agent | LangChain + LangGraph |
| Memory | ChromaDB |
| SSH | Paramiko |
| LLM | DeepSeek V4 Flash |
| MCP | Custom MCP Server |

### Features

- **Server Management** — Three-source instances, GPU monitoring, smart probing
- **Master Agent** — 8 LangChain Tools, LangGraph ReAct, streaming SSE
- **One-Click CC Deploy** — SSH → Node.js → Claude Code → tmux
- **Shared Memory** — ChromaDB with REST API for cross-agent access
- **MCP Server** — 5 Tools + 3 Resources, standard protocol
- **Knowledge Base** — Full-text search, category filters, per-agent memory
- **Cost Tracking** — AutoDL + DeepSeek dual balance

### License

MIT
