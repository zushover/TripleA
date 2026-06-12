# Triple A v0.1

<p align="center">
  <b>人与 AI 科研之间的第一道桥梁。</b><br>
  一句话租卡、用卡、跑实验。<br>
  一个面板搞定 GPU 管理、Agent 协作、实验记录、知识归档。<br>
  <b>不用打开服务器终端。</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-v0.1_active-1c1c1e?style=flat-square" />
  <img src="https://img.shields.io/badge/License-MIT-yellow?style=flat-square" />
  <img src="https://img.shields.io/badge/Python-3.12-blue?style=flat-square" />
  <img src="https://img.shields.io/badge/Vue-3.5-green?style=flat-square" />
</p>

---

## 这是什么

**Triple A 是面向 AI 科研工作者的智能实验助手。** 不是 GPU 管理工具，不是聊天机器人——它是一个能理解你研究意图、自主操作 GPU 服务器、管理实验全流程、归档知识记忆的 AI Agent 系统。

**核心理念：你不应该花时间操作服务器。你应该花时间思考。**

```
现在：                         有了 Triple A：
                               
ssh -p 49200 root@xxx          "在 3080Ti 上跑 LLaMA-Factory LoRA 微调"
nvidia-smi                      ↓
tmux new -s train              Agent 自主完成：
python train.py                   1. 检查 GPU 可用性
tail -f nohup.out                 2. 探测服务器环境
scp results back                  3. 远程执行训练脚本
手动写实验报告                    4. 实时监控 loss + GPU 利用率
                                 5. 异常自动恢复
                                 6. 结果收集 + 知识库归档
                                 7. 生成实验报告
                               
你只需要下一条指令。              剩下的 Triple A 全自动完成。
```

---

## 为什么叫 Triple A

**AutodlAgents Architecture** — 三层 Agent 架构：

```
A¹  Master Agent (面板端)       — 理解意图、分解任务、分发执行
A²  Sub-Agent (服务器端)         — 代码编写、实验运行、日志分析  
A³  Memory Agent (ChromaDB)     — 对话记忆、知识归档、决策审计
```

三者通过 MCP 协议和 REST API 通信，共享同一份 ChromaDB 记忆。Master 不执行代码，Sub-Agent 不自行决策，Memory 不遗忘。

---

## 架构

```
┌═══════════════════════════════════════════════════════════════┐
║                    Triple A 桌面端                            ║
║                                                               ║
║  ┌─────────────────────────────────────────────────────────┐ ║
║  │            A¹ Master Agent (LangGraph ReAct)            │ ║
║  │                                                         │ ║
║  │  理解意图 → 分解任务 → 匹配资源 → 分发执行 → 汇总归档    │ ║
║  │  8 个 Tool · DeepSeek V4 Flash · 流式 SSE 输出          │ ║
║  └───────────────────────┬─────────────────────────────────┘ ║
║                          │                                   ║
║  ┌───────────────────────┴─────────────────────────────────┐ ║
║  │              A³ Memory Hub (ChromaDB)                   │ ║
║  │  对话历史 · 实验记录 · 论文文档 · 决策审计               │ ║
║  │  全向量索引 · REST API 读写 · 语义搜索                   │ ║
║  └───────────────────────┬─────────────────────────────────┘ ║
║                          │                                   ║
║  ┌───────────────────────┴─────────────────────────────────┐ ║
║  │              MCP Server (标准化工具层)                   │ ║
║  │  GPU Monitor · Knowledge Base · Task Queue              │ ║
║  └───────────────────────┬─────────────────────────────────┘ ║
║                          │                                   ║
║  ┌───────────────────────┴─────────────────────────────────┐ ║
║  │              Vue 3 面板 (5 个页面)                       │ ║
║  │  服务器 · Triple A · 知识库 · 费用 · 设置                │ ║
║  └─────────────────────────────────────────────────────────┘ ║
║                                                               ║
║  技术栈：Tauri 2 (Rust) + Vue 3 + Python FastAPI + SQLite    ║
╚═══════════════════════════════════════════════════════════════╝
                           │
                    SSH + REST + MCP
                           │
┌──────────────────────────┴───────────────────────────────────┐
║                     GPU 服务器集群                            ║
║                                                               ║
║  ┌─────────────────────┐  ┌─────────────────────┐            ║
║  │  A² 3080Ti Server   │  │  A² 4090D Server    │            ║
║  │                     │  │                     │            ║
║  │  Claude Code (tmux) │  │  Claude Code (tmux) │            ║
║  │  Watchdog (任务轮询) │  │  Watchdog (任务轮询) │            ║
║  │                     │  │                     │            ║
║  │  ☀️ 一键部署         │  │  ☀️ 一键部署         │            ║
║  └─────────────────────┘  └─────────────────────┘            ║
║                                                               ║
║  Agent 可在服务器本地：写代码、装环境、跑实验、读日志、画图   ║
╚═══════════════════════════════════════════════════════════════╝
```

### 一次完整的科研实验流程

```
研究员 → Triple A 面板: "在 3080Ti 上跑 LLaMA-Factory 的 LoRA 微调，用 custom_qa_10k 数据集，预算 20 元"

A¹ Master Agent:
  Step 1 — 理解意图: experiment 类型，需要 GPU 服务器
  Step 2 — 查资源: 3080Ti 在线，12GB显存可用，余额充足
  Step 3 — 查历史: 知识库里搜到上次 LoRA 实验，最优 lr=2e-4
  Step 4 — 分解任务:
    Task 1: 环境检测 (CUDA/PyTorch/LLaMA-Factory)
    Task 2: 代码准备 (Git clone + LoRA 配置)
    Task 3: 数据上传 (custom_qa_10k)
    Task 4: 启动训练 (tmux + nohup)
    Task 5: 监控 (每120s 采集 GPU + loss)
    Task 6: 结果收集 (metrics + loss曲线)
    Task 7: 知识归档 (实验报告 → ChromaDB)
  Step 5 — 分发给 A² 执行
  Step 6 — 监控进度，异常时自动干预
  Step 7 — 汇总结果，生成报告

A² Sub-Agent (3080Ti 上的 Claude Code):
  → 收到 Task 1-4 → 本地执行（不需要每次 SSH）
  → 每 120s 向 A³ 上报心跳 + GPU 状态
  → OOM 了？自动减半 batch_size 重试
  → 训练完成 → 上传 metrics + loss.png → 通知 A¹

A³ Memory:
  → 实验配置: {"gpu":"3080Ti", "dataset":"custom_qa_10k", "lr":2e-4}
  → 实验结果: {"final_loss":0.08, "best_bleu":35.2, "cost":6.4}
  → 下次实验前 Master 自动检索参考

研究员收到通知: "实验完成 ✅ Loss 0.23→0.08, 耗时 3.2h, 花费 ¥6.4, 报告已归档"
```

---

## 功能

### 🖥 一句话管理 GPU

全部是自然语言。不需要记 SSH 命令、不需要配环境变量：

- **"列出所有 GPU 实例"** → 表格展示，状态/型号/单价一目了然
- **"检查 3080Ti 的利用率"** → SSH 直连采集 nvidia-smi 数据
- **"关掉所有空闲的 GPU"** → Agent 先预览，你确认后执行
- **"现在余额多少"** → AutoDL + DeepSeek 双余额

### 🤖 Master Agent 自主决策

8 个 LangChain Tool，LangGraph ReAct 循环。Agent 自己决定调哪个工具、传什么参数、怎么处理异常。不是 if/else 规则引擎——是真的 LLM 推理。

| Tool | 做什么 | 
|------|--------|
| `list_gpu_instances` | 列出所有实例 | 
| `check_gpu_utilization(uuid)` | 实时 GPU 利用率/显存/温度 | 
| `probe_instance_health(uuid)` | SSH 探测 + GPU 型号 + 系统信息 | 
| `execute_on_server(uuid, cmd)` | 远程执行任意命令 | 
| `delegate_to_server(uuid, task)` | 委派自然语言任务给服务器 CC | 
| `get_balance_and_cost` | AutoDL 余额 + 消费统计 | 
| `shutdown_idle_instance(uuid)` | 安全关机（需确认）| 
| `save_to_knowledge_base(cat, title, content)` | 自动归档实验数据 | 

### ☀️ 一键部署 Claude Code

这是 Triple A 最独特的能力。点一下太阳按钮：

```
SSH 连接 → 检测环境 → 装 Node.js → npm install claude-code
→ SFTP 配置 DeepSeek API → tmux 启动 CC + Watchdog
→ 大概 2 分钟 → 服务器上多了一个能自主写代码跑实验的 Agent
```

部署完后，你可以 `delegate_to_server(uuid, "帮我分析训练日志找出 loss 不收敛的原因")` —— 服务器端的 CC 在本地读文件、写分析、返回结论。不需要你把几百 MB 的日志下载到本地。

### 🧠 永不遗忘的知识库

ChromaDB 向量数据库——所有对话、实验、决策自动归档：

- **语义搜索：** "上次 LoRA 微调用的什么学习率？" → 直接搜到
- **分类浏览：** 对话 / 实验 / 决策 / 文档
- **Agent 专属记忆：** 每个 Agent 的对话和实验单独归档
- **循环检测：** 相似度 >90% 的重复决策自动告警

### 📡 MCP Server

GPU 监控以 MCP 标准协议暴露。任何支持 MCP 的工具（Claude Code、Cursor、自定义 Agent）都能发现并调用你的 GPU 数据。

### 💰 双余额实时追踪

AutoDL 服务器费用 + DeepSeek API 费用，一个面板看全部。余额不足自动推送告警。

---

## 技术栈

| 层 | 技术 | 为什么选它 |
|---|------|----------|
| 桌面壳 | Tauri 2 (Rust) | 比 Electron 轻 20 倍，原生系统托盘 |
| 前端 | Vue 3 + TypeScript | 响应式、类型安全 |
| 后端 | Python FastAPI | 异步、自动文档、生态好 |
| Agent | LangChain + LangGraph | ReAct 循环、Tool Calling、流式输出 |
| 记忆 | ChromaDB | 零配置向量数据库、Python 原生 |
| SSH | Paramiko | 纯 Python SSH、SFTP 支持 |
| LLM | DeepSeek V4 Flash | 兼容 Anthropic API、成本低、速度快 |
| 数据库 | SQLite (WAL) | 零配置、适合桌面应用 |
| MCP | 自建 Server | 标准协议、跨工具互通 |
| 打包 | PyInstaller + NSIS | 单文件 exe、无需安装 Python |

---

## 快速开始

### 你需要

- Python 3.12+
- Node.js 20+（前端开发用）
- AutoDL 账号（或任意有 SSH 的 GPU 服务器）
- DeepSeek API Key（[platform.deepseek.com](https://platform.deepseek.com)）

### 3 分钟跑起来

```bash
git clone https://github.com/zushover/TripleA.git
cd TripleA

# 后端
pip install -r requirements.txt
cp config.example.yaml config.yaml
# 编辑 config.yaml：填入 AutoDL Token 和 DeepSeek API Key
python sidecar.py

# 前端（新终端）
npm install && npm run dev

# 打开浏览器 → http://127.0.0.1:8899
```

### 注册一台 GPU 服务器

```
1. 打开面板 → 服务器 → 注册
2. 粘贴 AutoDL 的 SSH 连接命令: ssh -p 49200 root@connect.xxx
3. 填入 SSH 密码
4. 点探测 → 自动识别 GPU 型号
5. 点 ☀️ 部署 CC → 等 2 分钟
6. 服务器上多了一个 AI Agent
```

### 然后你只需要说话

```
"检查 GPU 状态"          → Agent 列实例 + 查利用率
"在 3080Ti 上跑个实验"   → Agent 写代码 + 启动训练 + 监控 + 归档
"上次 LoRA 最优参数是啥"  → 知识库语义搜索
"关机空闲实例"           → 预览 → 确认 → 执行
```

---

## 实测

所有功能经真实 GPU 服务器验证：

- GPU 探测 → 实时识别 3080Ti/4090D ✅
- Agent ReAct → 8 个 Tool 全部调通 ✅
- ☀️ 一键部署 → 成功部署到远程服务器 ✅
- 知识库 → ChromaDB 存储 + 语义搜索 ✅
- MCP → 标准协议 tools/resources 调用 ✅
- 费用 → AutoDL + DeepSeek 双余额 ✅
- 流式输出 → SSE 逐 token 渲染 ✅

---

## 项目结构

```
TripleA/
├── autodl_manager/agent/    # 🔥 Agent 核心模块
│   ├── tools.py             #   8 个 LangChain Tool
│   ├── agent_loop.py        #   LangGraph ReAct 循环
│   ├── memory.py            #   ChromaDB 记忆层
│   ├── mcp_server.py        #   MCP GPU Monitor Server
│   ├── deploy.py            #   ☀️ 一键部署 Claude Code
│   ├── watchdog.py          #   服务器端任务监听器
│   ├── executor.py          #   A² 执行器
│   ├── multi_agent.py       #   多 Agent 编排器
│   └── prompts.py           #   System Prompt
├── src/                     # Vue 3 前端
│   ├── App.vue              #   根组件（状态中枢）
│   └── components/
│       ├── Dashboard.vue    #   服务器管理
│       ├── AgentLog.vue     #   Triple A 对话
│       ├── KnowledgeBase.vue#   知识库
│       └── CostAnalysis.vue #   费用分析
├── src-tauri/               # Tauri 桌面壳
├── sidecar.py               # Python sidecar 入口
└── config.example.yaml      # 配置模板
```

---

## 路线图

```
v0.1 ✅  当前：Master Agent + 服务器管理 + 一键部署 CC + 知识库 + MCP
v0.2 🔜  多 Agent 实时协作（Watchdog 激活，A² 全自主运行）
v0.3 📋  Agent 评估体系（成功率 / Token 效率 / 时间成本）
v0.4 📋  Tauri 打包发布（Windows .exe / macOS .dmg）
v0.5 📋  团队协作（多人共享同一知识库，实验对比分析）
```

---

## FAQ

**浏览器测试和打包后是一样的吗？**

是的。浏览器跑的是 Vite 开发服务器，Tauri 打包后 WebView 加载的是同一份 Vue 代码构建的 `dist/` 静态文件。Python sidecar 在两种模式下都是独立进程。开发和生产的区别只是壳。

**需要公网 IP 吗？**

不需要。Master Agent 在你电脑上，通过 SSH 直连服务器。服务器端子 Agent 通过同样的 SSH 通道通信。不需要额外的公网端口。

**支持 OpenAI API 吗？**

支持任何 OpenAI 兼容的 API。在 `config.yaml` 中修改 `llm.api_base` 和 `llm.model` 即可。

---

## License

MIT

---

<p align="center">
  <sub>Built by a researcher, for researchers. 让 AI 替你操作服务器，你只负责思考。</sub>
</p>

---

# Triple A v0.1

<p align="center">
  <b>The first bridge between human researchers and AI-powered scientific computing.</b>
</p>

Triple A is an AI agent system for ML researchers. It manages GPU servers, runs experiments, archives knowledge — all through natural language. You never open a terminal.

**One sentence to rent GPUs, run experiments, archive results.**

Built with LangGraph ReAct Agent (8 Tools), MCP Server, ChromaDB memory, one-click Claude Code deployment. Tauri 2 + Vue 3 + Python FastAPI.

### Quick Start

```bash
git clone https://github.com/zushover/TripleA.git && cd TripleA
pip install -r requirements.txt && cp config.example.yaml config.yaml
python sidecar.py           # Backend :8899
npm install && npm run dev  # Frontend dev server
# Open http://127.0.0.1:8899
```

### Architecture

A¹ Master Agent (LangGraph ReAct) → A² Sub-Agent (Claude Code on GPU servers) → A³ Memory (ChromaDB). Communication via MCP + REST + SSH.

### License

MIT
