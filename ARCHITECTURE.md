# Hermes Agent 系统架构 Wiki

> 基于源码分析生成 · 版本对应 `upstream/main` @ `424e9f36` · 2026-04-20

---

## 目录

1. [项目定位与技术栈](#1-项目定位与技术栈)
2. [目录结构与模块划分](#2-目录结构与模块划分)
3. [核心子系统](#3-核心子系统)
   - 3.1 [Agent 核心（LLM 适配层）](#31-agent-核心llm-适配层)
   - 3.2 [消息网关与平台适配器](#32-消息网关与平台适配器)
   - 3.3 [工具系统](#33-工具系统)
   - 3.4 [技能系统](#34-技能系统)
   - 3.5 [内存 / 记忆系统](#35-内存--记忆系统)
   - 3.6 [插件系统](#36-插件系统)
   - 3.7 [调度 / Cron 系统](#37-调度--cron-系统)
   - 3.8 [Terminal 后端](#38-terminal-后端)
   - 3.9 [ACP 适配器](#39-acp-适配器)
   - 3.10 [Honcho 集成](#310-honcho-集成)
   - 3.11 [Atropos / RL 训练系统](#311-atropos--rl-训练系统)
4. [数据流与关键交互](#4-数据流与关键交互)
   - 4.1 [CLI 模式完整消息流](#41-cli-模式完整消息流)
   - 4.2 [网关模式消息流（Telegram 示例）](#42-网关模式消息流telegram-示例)
   - 4.3 [工具执行流程](#43-工具执行流程)
   - 4.4 [会话持久化与恢复](#44-会话持久化与恢复)
5. [配置系统](#5-配置系统)
6. [持久化与存储](#6-持久化与存储)
7. [关键特性](#7-关键特性)
8. [架构设计模式总结](#8-架构设计模式总结)

---

## 1. 项目定位与技术栈

**Hermes Agent** 是由 [Nous Research](https://nousresearch.com) 开发的**自我改进 AI Agent 框架**，核心特点：

- **模型无关**：支持 200+ 模型（Anthropic、OpenAI、Google、Meta、Mistral 等，通过 OpenRouter 聚合）
- **多平台网关**：Telegram、Discord、Slack、WhatsApp、Signal、Email、Matrix 等 20+ 消息平台
- **自学习循环**：从经验中自动创建技能、跨会话记忆、FTS5 历史搜索
- **可插拔架构**：工具、内存提供者、平台适配器均可扩展
- **企业级特性**：多租户配置文件、命令审批、PII 移除、成本追踪

### 核心技术栈

| 类别 | 主要依赖 |
|------|---------|
| 语言 | Python 3.11+ |
| LLM SDK | `anthropic>=0.39.0`, `openai>=2.21.0` |
| 验证 | `pydantic>=2.12.5` |
| HTTP | `httpx[socks]>=0.28.1` |
| 配置 | `pyyaml>=6.0.2`, `jinja2>=3.1.5` |
| CLI | `fire>=0.7.1`, `rich>=14.3.3`, `prompt_toolkit` |
| 消息平台 | `python-telegram-bot>=22.6`, `discord.py>=2.7.1`, `slack-bolt>=1.18.0` |
| 计算后端 | `modal>=1.0.0`, `daytona>=0.148.0` |
| 存储 | SQLite（WAL 模式 + FTS5） |
| 可选 | `honcho-ai>=2.0.1`, `mcp>=1.2.0`, `faster-whisper>=1.0.0` |

---

## 2. 目录结构与模块划分

```
hermes-agent/
│
├── run_agent.py              # AIAgent 主类（核心循环，~12,000 行）
├── cli.py                    # 交互式 CLI（prompt_toolkit TUI）
├── model_tools.py            # 工具调度中介
├── toolsets.py               # 工具集定义与组合
├── batch_runner.py           # 并行批处理 / 轨迹生成
├── rl_cli.py                 # 强化学习数据生成 CLI
│
├── agent/                    # LLM 适配层与 Agent 核心（~21,000 行）
│   ├── anthropic_adapter.py       # Anthropic Messages API 适配器
│   ├── bedrock_adapter.py         # AWS Bedrock 支持
│   ├── gemini_native_adapter.py   # Google Gemini 原生适配
│   ├── gemini_cloudcode_adapter.py
│   ├── auxiliary_client.py        # 辅助模型（摘要、成本估算）
│   ├── context_compressor.py      # 上下文窗口自动压缩
│   ├── context_engine.py          # 上下文路由
│   ├── credential_pool.py         # 多账号凭证管理
│   ├── memory_manager.py          # 内存管理（内置 + 插件）
│   ├── memory_provider.py         # 内存提供者接口
│   ├── prompt_builder.py          # 系统提示动态组装
│   ├── prompt_caching.py          # Anthropic 提示缓存
│   ├── model_metadata.py          # 模型元数据与上下文限制
│   ├── skill_utils.py             # 技能元数据工具
│   ├── trajectory.py              # 轨迹保存与格式化
│   ├── insights.py                # 使用统计与洞察
│   └── ...
│
├── gateway/                  # 消息网关（20+ 平台）
│   ├── run.py                     # 主网关运行器（AsyncIO 事件循环）
│   ├── session.py                 # 会话管理
│   ├── delivery.py                # 消息投递
│   ├── pairing.py                 # DM 配对认证
│   ├── hooks.py                   # 事件钩子系统
│   ├── mirror.py                  # 消息镜像（多平台转发）
│   └── platforms/                 # 平台适配器
│       ├── base.py                    # 基础接口
│       ├── telegram.py / discord.py / slack.py
│       ├── whatsapp.py / signal.py / matrix.py
│       ├── email.py / sms.py / mattermost.py
│       ├── homeassistant.py / api_server.py
│       ├── dingtalk.py / feishu.py / wecom.py / weixin.py
│       └── qqbot/ / bluebubbles.py
│
├── tools/                    # 70+ 工具实现（~70,000 行）
│   ├── registry.py                # 工具注册表
│   ├── web_tools.py               # Web 搜索 & 提取
│   ├── terminal_tool.py           # 终端执行（6 个后端）
│   ├── browser_tool.py            # 浏览器自动化
│   ├── file_tools.py              # 文件读写
│   ├── vision_tools.py            # 图像分析
│   ├── image_generation_tool.py   # 图像生成
│   ├── tts_tool.py / transcription_tools.py
│   ├── memory_tool.py             # 持久记忆操作
│   ├── session_search_tool.py     # 跨会话历史搜索
│   ├── cronjob_tools.py           # Cron 作业管理
│   ├── delegate_tool.py           # 子代理委托
│   ├── mcp_tool.py                # MCP 协议支持
│   └── ...
│
├── skills/                   # 27 个内置技能目录
│   ├── creative/              # 图像生成、创意写作
│   ├── github/                # GitHub 集成
│   ├── research/              # 数据收集与分析
│   ├── software-development/  # 编程、DevOps
│   ├── productivity/          # 生产力工具
│   ├── data-science/          # ML、统计
│   ├── red-teaming/           # 安全测试
│   ├── mcp/                   # MCP 服务器管理
│   └── ...
│
├── plugins/                  # 可插拔扩展
│   ├── memory/
│   │   ├── honcho/            # Honcho AI 用户建模
│   │   ├── mem0/              # Mem0 向量内存
│   │   ├── holographic/       # 混合检索
│   │   └── ...
│   └── context_engine/
│
├── cron/                     # Cron 调度系统
│   ├── jobs.py
│   └── scheduler.py
│
├── acp_adapter/              # Agent Client Protocol 适配器
│
├── hermes_cli/               # CLI 主程序（40+ 文件）
│   ├── main.py / commands.py
│   ├── config.py              # 配置管理
│   ├── gateway.py / setup.py / doctor.py
│   ├── profiles.py            # 多租户
│   ├── mcp_config.py / plugins_cmd.py
│   └── web_server.py          # Web 仪表盘
│
├── hermes_state.py           # SQLite 会话数据库（WAL 模式）
├── hermes_constants.py       # 全局常量
├── hermes_logging.py         # 结构化日志
├── trajectory_compressor.py  # 轨迹压缩
├── mcp_serve.py              # MCP 服务器
│
├── environments/             # 多环境配置
├── docker/ / nix/ / scripts/ # 部署支持
└── tests/                    # 测试套件
```

---

## 3. 核心子系统

### 3.1 Agent 核心（LLM 适配层）

`run_agent.py` 中的 `AIAgent` 是整个系统的核心，负责：

- 管理与 LLM 的对话循环
- 工具调用调度与并行执行
- 上下文窗口管理（自动压缩）
- 内存注入与同步
- 流式响应与中断处理

**API 适配器抽象层：**

```
anthropic_adapter.py   → Anthropic Messages API（翻译 OpenAI 格式）
                          支持 OAuth、API 密钥、Claude Code 凭证
                          扩展思维（extended thinking）

bedrock_adapter.py     → AWS Bedrock（IAM 认证，区域感知）

gemini_*_adapter.py    → Google Gemini 原生 API + CloudCode

auxiliary_client.py    → 辅助 LLM（摘要、成本估算，廉价快速）
```

**上下文管理：**

| 文件 | 职责 |
|------|------|
| `context_compressor.py` | 检测上下文窗口饱和，执行无损摘要压缩（保护头/尾） |
| `context_engine.py` | 上下文路由与合成、引用追踪 |
| `prompt_builder.py` | 系统提示动态组装（身份、技能索引、记忆、平台提示） |
| `prompt_caching.py` | Anthropic 提示前缀缓存（缓存读取成本 = 10% 输入成本） |
| `credential_pool.py` | 多账号凭证 LRU 轮换，避免单账号限速 |

**迭代预算与安全机制：**

- 工具循环上限：90 次迭代
- 自动上下文窗口检测（`model_metadata.py`）
- 故障转移链：主模型失败 → `config.fallback` 模型 → 报错
- 熔断器：MCP 工具处理器防止重试烧干循环
- 流式中断：用户发送新消息可打断当前执行

---

### 3.2 消息网关与平台适配器

**核心组件：**

```
GatewayRunner (gateway/run.py)
  ├─ AsyncIO 事件循环
  ├─ 平台管理器（20+ 平台并行）
  ├─ 会话路由（session_key → AIAgent 缓存）
  ├─ AIAgent LRU 缓存（最多 128 实例）
  ├─ 会话过期监视器
  ├─ 平台重连监视器
  └─ 优雅关闭处理

BasePlatformAdapter (gateway/platforms/base.py)
  ├─ 抽象接口：send_text / send_image / send_file
  ├─ UTF-16 消息长度限制处理
  ├─ 媒体上传处理
  ├─ 用户认证与 DM 配对
  └─ 速率限制
```

**会话路由键格式：**

```
session_key = "{agent}:{context}:{platform}:{type}:{id}"
例：  "agent:main:telegram:dm:123456789"
      "agent:main:discord:channel:987654321"
      "agent:main:cli:local:default"
```

**支持的平台（20+）：**

| 平台 | 适配器文件 |
|------|-----------|
| Telegram | `telegram.py` |
| Discord | `discord.py` |
| Slack | `slack.py` |
| WhatsApp | `whatsapp.py` |
| Signal | `signal.py` |
| Matrix/Element | `matrix.py` |
| Email | `email.py` |
| SMS | `sms.py` |
| Mattermost | `mattermost.py` |
| Home Assistant | `homeassistant.py` |
| REST API | `api_server.py` |
| Webhook | `webhook.py` |
| DingTalk | `dingtalk.py` |
| Feishu/Lark | `feishu.py` |
| WeChat Work | `wecom.py` |
| WeChat 公众号 | `weixin.py` |
| QQ Bot | `qqbot/` |
| BlueBubbles | `bluebubbles.py` |

---

### 3.3 工具系统

**注册机制：**

```python
# tools/xxx.py 中，工具通过装饰器自注册：
registry.register(
    name="web_search",
    schema={...},         # OpenAI JSON schema
    handler=handler_fn,
    toolset="web",        # 工具集分组
    check_fn=lambda: API_KEY is not None,  # 可用性检查
    is_async=True,
    description="Search the web",
    emoji="🔍"
)
```

**工具调度流程（model_tools.py）：**

1. `get_tool_definitions(enabled_toolsets)` → 过滤并返回可用工具 schema
2. LLM 输出 `tool_use` → `handle_function_call(name, args)`
3. `registry.dispatch(name, args)` → 执行处理函数
4. 异步工具通过持久事件循环桥接（`_run_async`）
5. 并行执行通过 `ThreadPoolExecutor`
6. 结果截断（`max_result_size`）后返回给 LLM

**工具分类（70+ 个）：**

| 类别 | 工具 |
|------|------|
| Web & 信息 | `web_search`, `web_extract`, `session_search` |
| 代码 & 终端 | `terminal`（6 后端）, `execute_code`, `process` |
| 文件 | `read_file`, `write_file`, `patch`, `search_files` |
| 视觉 & 图像 | `vision_analyze`, `image_generate`, `browser_vision` |
| 浏览器 | `browser_navigate/click/type/scroll/cdp/console` |
| 媒体 | `text_to_speech`, `transcription` |
| 规划 & 记忆 | `todo`, `memory`, `clarify` |
| 技能 | `skills_list`, `skill_view`, `skill_manage` |
| 系统 | `delegate_task`, `cronjob`, `send_message`, `interrupt`, `approval` |
| 协议 | `mcp_tool`（Model Context Protocol） |
| 智能家居 | `ha_*`（Home Assistant） |

**并行执行策略：**

```
可并行（只读、无共享状态）：
  ✅ web_search, read_file, vision_analyze, terminal（独立路径）

顺序执行（有交互或破坏性）：
  ⛔ clarify（用户交互）
  ⛔ 破坏性命令检测到竞态
  ⛔ 路径重叠（多个写操作同一文件）
```

---

### 3.4 技能系统

**技能目录结构：**

```
skills/
└── category/
    └── skill-name/
        ├── SKILL.md              # 主指令（YAML 前置 + Markdown）
        ├── references/           # 支持文档
        │   ├── api.md
        │   └── examples.md
        ├── templates/            # 输出模板
        └── assets/               # 补充文件
```

**SKILL.md 前置元数据：**

```yaml
---
name: skill-name              # 必需，≤64 字符
description: 简述             # 必需，≤1024 字符
version: 1.0.0
platforms: [macos, linux]     # 可选，平台限制
prerequisites:
  env_vars: [API_KEY]
  commands: [curl, jq]
metadata:
  tags: [ai, ml]
  related_skills: [skill2]
---
# Skill Instructions
...
```

**渐进式披露架构（Token 优化）：**

| 层级 | 工具 | 内容 | Token 开销 |
|------|------|------|-----------|
| Tier 1 | `skills_list` | 名称 + 描述 | <1KB/技能 |
| Tier 2 | `skill_view` | 完整 SKILL.md | 200–2000 行 |
| Tier 3 | `skill_view --path` | 参考文档、模板 | 按需加载 |

**内置技能目录（27 个）：**

`creative` · `github` · `research` · `software-development` · `productivity` · `note-taking` · `email` · `media` · `smart-home` · `data-science` · `red-teaming` · `feeds` · `gaming` · `diagramming` · `mcp` · `autonomous-ai-agents` · `inference-sh` · 等

---

### 3.5 内存 / 记忆系统

**架构分层：**

```
MemoryManager (agent/memory_manager.py)
  │
  ├─ BuiltinMemoryProvider（始终存在）
  │   ├─ MEMORY.md（用户显式记忆，YAML 前置 + 自由格式）
  │   ├─ USER.md（用户档案）
  │   ├─ FTS5 会话搜索（跨会话 + LLM 摘要重排）
  │   └─ memory_tool（CRUD 工具）
  │
  └─ 外部插件提供者（最多 1 个）
      ├─ Honcho AI（方言法用户建模）
      ├─ Mem0（向量内存）
      ├─ Holographic（混合检索）
      ├─ SuperMemory / RetainDB / OpenViking / Hindsight
      └─ ...
```

**内存注入时序：**

```
Turn 开始
  ↓
prefetch_all(user_msg) → 拉取相关记忆
  ↓
系统提示注入（围栏标签防注入）：
  <memory-context>
  [System note: The following is recalled memory...]
  {memory content}
  </memory-context>
  ↓
LLM 处理
  ↓
sync_all(user_msg, assistant_resp) → 同步/更新记忆
  ↓
Turn 结束
```

**内置记忆存储位置：**

```
~/.hermes/
├─ MEMORY.md        # 用户记忆（持久）
├─ USER.md          # 用户档案（持久）
└─ state.db         # 会话消息（FTS5 索引）
```

---

### 3.6 插件系统

**加载机制：**

启动时扫描 `plugins/*/` 目录，导入 `__init__.py`，注册钩子和提供者。

**生命周期钩子：**

| 钩子 | 触发时机 |
|------|---------|
| `on_session_start` | 新会话第一个 turn |
| `on_prefetch` | 内存提取前 |
| `on_system_prompt` | 系统提示构建后 |
| `on_turn_start` | API 调用前 |
| `on_tool_call` | 工具执行前 |
| `on_tool_result` | 工具完成后 |
| `on_turn_end` | turn 完成后 |
| `on_session_end` | 会话关闭 |

**内存插件接口：**

```python
class MemoryProvider:
    async def get_tool_definitions() → Dict[str, Schema]
    async def prefetch(user_message: str) → str
    async def sync(user_msg: str, assistant_resp: str) → None
    def build_system_prompt() → str
```

> 插件故障不会终止 Agent，仅记录警告日志，优雅降级。

---

### 3.7 调度 / Cron 系统

**作业定义格式（`~/.hermes/cron/jobs.json`）：**

```json
{
  "id": "daily-report",
  "schedule": "0 9 * * *",
  "prompt": "Generate daily report",
  "deliver": "telegram",
  "deliver_target": "12345",
  "enabled": true,
  "paused": false
}
```

**执行流程：**

```
GatewayRunner.tick()（每 60 秒）
  ↓
scheduler.get_ready_jobs()
  ↓
为每个就绪 job：
  ├─ 文件锁（防止重复执行）
  ├─ 创建隔离 AIAgent 实例（无历史记录）
  ├─ 执行 prompt
  └─ delivery.send_response(platform, target, result)
```

**特性：**
- 标准 cron 表达式支持
- 文件锁防重复执行
- 指数退避自动重试
- 执行日志持久化（`~/.hermes/cron/runs/`）

---

### 3.8 Terminal 后端

**6 个后端的抽象接口：**

```python
class TerminalEnvironment:
    def execute(cmd: str) → Tuple[returncode, stdout, stderr]
    def is_available() → bool
    def setup() → None
    def cleanup() → None
```

| 后端 | 文件 | 适用场景 |
|------|------|---------|
| Local | `local.py` | 本机 subprocess |
| Docker | `docker.py` | 容器隔离 |
| SSH | `ssh.py` | 远程服务器 |
| Modal | `modal.py` | Serverless GPU/CPU |
| Daytona | `daytona.py` | 云开发环境 |
| Singularity | `singularity.py` | HPC 集群 |

**后端选择策略：**

1. `config.yaml` 中的 `terminal.backend` 首选项
2. `is_available()` 可用性检查
3. 回退链（依赖缺失时自动降级）

**持久环境（Modal、Daytona）：**
- VM 空闲时休眠，turn 开始时自动唤醒
- 跨会话状态保留
- 按分钟计费，成本最低

---

### 3.9 ACP 适配器

**Agent Client Protocol 集成（`acp_adapter/`）：**

```
ACP Client
  ↓ (JSON-RPC via stdio)
HermesACPAgent (acp_adapter/server.py)
  ├─ initialize()
  ├─ process_message()
  └─ tool_call()
      ↓
  AIAgent.run_conversation()
      ↓
  JSON-RPC response → ACP Client
```

**核心文件：**

| 文件 | 职责 |
|------|------|
| `server.py` | ACP 服务器（`HermesACPAgent` 类） |
| `session.py` | 会话状态管理 |
| `auth.py` | OAuth & 令牌验证 |
| `permissions.py` | 权限边界 |
| `tools.py` | ACP 工具定义映射 |
| `events.py` | 事件处理 |

---

### 3.10 Honcho 集成

**功能：**
- 向量化存储用户交互
- 构建用户档案（信念、偏好）
- 方言法（Dialectics）：信念进化追踪
- 隐含表示学习

**集成点：**

```
plugins/memory/honcho/
  ├─ __init__.py    → 注册为内存插件
  ├─ client.py      → Honcho API 客户端
  ├─ session.py     → 会话 ID 管理
  └─ cli.py         → CLI 集成

注入流程：
  build_system_prompt() → 用户洞察注入系统提示
  sync()               → turn 结束后更新用户档案
```

---

### 3.11 Atropos / RL 训练系统

**批量轨迹生成（`batch_runner.py`）：**

```
JSONL 数据集
  ↓
ThreadPoolExecutor（并行工作进程）
  ↓
每个样本 → AIAgent.run_conversation()
  ↓
轨迹规范化（trajectory_compressor.py）
  ↓
ShareGPT 兼容格式输出：
  {
    "conversations": [
      {"from": "user", "value": "..."},
      {"from": "assistant", "value": "..."}
    ],
    "timestamp": "...",
    "model": "...",
    "completed": true
  }
```

**特性：**
- 检查点恢复（中断续跑）
- 使用统计聚合
- HuggingFace 数据集兼容
- SWE-bench 风格评估（`mini_swe_runner.py`）

---

## 4. 数据流与关键交互

### 4.1 CLI 模式完整消息流

```
用户输入 "hello"
  │
  ▼ (prompt_toolkit)
cli.py::_chat_loop()
  ├─ 从 SessionDB 加载历史消息
  └─ 调用 AIAgent.run_conversation()
      │
      ├─ Step 1: messages.append({"role": "user", "content": "hello"})
      │
      ├─ Step 2: _memory_manager.prefetch_all()
      │              └─ 返回 <memory-context>...</memory-context>
      │
      ├─ Step 3: _build_system_prompt()
      │              ├─ 身份/角色（SOUL.md、AGENTS.md）
      │              ├─ 平台提示（CLI 模式）
      │              ├─ 记忆指导
      │              ├─ 技能索引（懒加载）
      │              └─ 上下文文件（.cursorrules 等）
      │
      ├─ Step 4: get_tool_definitions(enabled_toolsets) → [schema, ...]
      │
      ├─ Step 5: _interruptible_api_call()
      │              ├─ 提示缓存应用（Anthropic 前缀缓存）
      │              ├─ 选择适配器（anthropic / bedrock / gemini）
      │              └─ 发送请求，流式接收响应
      │
      ├─ Step 6: 工具调用循环（while tool_calls）
      │              ├─ 解析 tool_use 块
      │              ├─ 并行或顺序执行工具
      │              │   └─ registry.dispatch(name, args) → result
      │              ├─ messages.append(tool_result)
      │              ├─ 检查迭代预算（≤90）
      │              └─ 发送下一次 API 请求
      │
      ├─ Step 7: 响应投递
      │              ├─ 移除 <think> 推理标签
      │              └─ 流式打印到 TUI（语法高亮）
      │
      └─ Step 8: 持久化
                     ├─ SessionDB.save_session()
                     ├─ SessionDB.save_messages()
                     ├─ 成本估算与保存
                     └─ _memory_manager.sync_all()
```

### 4.2 网关模式消息流（Telegram 示例）

```
用户发消息给 @HermesBot
  │
  ▼ (Telegram 服务器)
TelegramAdapter.handle_update(update)
  │
  ▼
GatewayRunner._handle_message(event)
  ├─ 命令检查（/new, /model, /stop 等）
  ├─ session_key = "agent:main:telegram:dm:{user_id}"
  ├─ 查询或创建 Session（SessionDB）
  ├─ 获取或创建 AIAgent（LRU 缓存，最多 128 个）
  │
  └─ AIAgent.run_conversation(callback=stream_delta)
      │（同 CLI 流程，附加状态回调）
      │
      ▼
delivery.send_response(platform="telegram", ...)
  ├─ 长消息分割（UTF-16 ≤ 4096 字符）
  └─ TelegramAdapter.send_text(chat_id, text)
      │
      ▼
消息出现在 Telegram 聊天中
```

### 4.3 工具执行流程

```
LLM 输出：
  {
    "type": "tool_use",
    "id": "toolu_0123...",
    "name": "web_search",
    "input": {"query": "latest AI news"}
  }
  │
  ▼
AIAgent._process_tool_calls()
  │
  ├─ 并行判断（_should_parallelize_tool_batch）
  │   ├─ 只读工具？ ✅ → ThreadPoolExecutor
  │   ├─ 无共享状态？
  │   └─ 无用户交互工具？
  │
  ├─ 并行路径：
  │   └─ handle_function_call(name, args)
  │       ├─ registry.dispatch("web_search", args)
  │       ├─ 执行处理函数（异步桥接）
  │       ├─ 结果截断（max_result_size）
  │       └─ 返回 result_str
  │
  └─ 顺序路径（clarify / 破坏性命令）：
      └─ 逐一执行，等待用户确认
  │
  ▼
messages.append({
  "type": "tool_result",
  "tool_use_id": "toolu_0123...",
  "content": "[search results...]"
})
  │
  ▼
发送下一次 API 请求（包含工具结果）
```

### 4.4 会话持久化与恢复

**SQLite Schema（`hermes_state.py`）：**

```sql
-- 会话表
sessions (
  id, source, user_id_hash, model,
  system_prompt,          -- 缓存，用于提示前缀缓存
  parent_session_id,      -- 压缩时链接前序会话
  started_at, ended_at,
  message_count, tool_call_count,
  input_tokens, output_tokens,
  cache_read_tokens, cache_write_tokens,
  estimated_cost_usd,
  title                   -- AI 生成的会话标题
)

-- 消息表
messages (
  id, session_id,
  role,                   -- user / assistant / system
  content,
  tool_calls,             -- JSON（工具调用历史）
  tool_name, tool_call_id,
  timestamp, token_count,
  finish_reason,
  reasoning,              -- 扩展思维内容
  reasoning_details       -- 结构化推理
)

-- 全文搜索虚拟表（FTS5）
messages_fts → 触发器同步，支持 session_search 工具
```

**恢复流程：**

```
1. 用户发送消息
2. GatewayRunner.get_or_create_session(session_key)
3. SessionDB.get_session() → 加载会话元数据
4. SessionDB.get_messages(session_id) → 加载历史消息
5. AIAgent(conversation_history=messages)
6. 加载缓存的 system_prompt（跳过重建，节省 Token）
7. run_conversation() 从历史记录继续
```

---

## 5. 配置系统

### 配置优先级（高 → 低）

```
1. 环境变量（HERMES_*）
2. ~/.hermes/config.yaml（用户配置）
3. ~/.hermes/profiles/<name>/config.yaml（配置文件特定）
4. 内置默认值（hermes_cli/config.py）
```

### 核心配置分类

| 前缀 | 职责 |
|------|------|
| `model.*` | LLM 提供商、模型 ID、参数 |
| `tools.*` | 启用的工具集 |
| `memory.*` | 内存提供者配置 |
| `gateway.*` | 平台、DM 配对、重置策略 |
| `plugins.*` | 插件配置 |
| `cron.*` | 调度设置 |
| `terminal.*` | 后端选择、环境变量 |
| `llm.*` | 上下文限制、缓存、推理 |
| `security.*` | 命令审批、配对 |
| `display.*` | 终端输出格式 |
| `network.*` | 代理、IPv4 强制 |

### 配置管理命令

```bash
hermes config set model.provider anthropic
hermes config get model.model
hermes config reset
hermes config validate
```

---

## 6. 持久化与存储

### 目录布局

```
~/.hermes/
├─ config.yaml                # 主配置
├─ .env                       # API 密钥和秘密
├─ state.db                   # SQLite（会话 + 消息 + FTS5）
├─ state.db-wal               # Write-Ahead Log
├─ state.db-shm               # 共享内存
│
├─ MEMORY.md                  # 用户显式记忆
├─ USER.md                    # 用户档案
│
├─ cache/
│   ├─ images/                # 图像生成缓存
│   ├─ models/                # 下载模型（Whisper 等）
│   └─ embeddings/            # 向量嵌入
│
├─ skills/                    # 用户创建的技能
│
├─ profiles/                  # 多租户配置
│   ├─ default/
│   │   ├─ config.yaml
│   │   └─ state.db
│   └─ work/
│
├─ cron/
│   ├─ jobs.json              # Cron 作业定义（JSONL）
│   └─ runs/                  # 执行日志
│
├─ sessions/
│   └─ sessions.json          # 会话路由表
│
└─ logs/
    └─ *.log                  # 结构化日志
```

### SQLite 设计亮点

| 特性 | 实现 |
|------|------|
| WAL 模式 | 多读单写并发，零死锁 |
| FTS5 全文搜索 | 触发器同步，`session_search` 工具依赖 |
| 会话链 | `parent_session_id` 链接压缩前的前序会话 |
| 原子写入 | `atomic_yaml_write()` 防止配置文件损坏 |
| 应用级重试 | 指数抖动退避，避免写锁车队效应 |
| 成本追踪 | 精细到每个 turn 的 Token 用量 |

---

## 7. 关键特性

### 7.1 提示缓存（Anthropic）

- 系统提示、技能索引、上下文文件标记 `"cache_control": {"type": "ephemeral"}`
- 缓存读取 Token 成本 = 10% 标准输入成本
- 跨 turn 持续有效（TTL ~5 分钟）
- 最小缓存单元：1024 Token

### 7.2 扩展思维 / 推理模式

| 配置 | 适用模型 | Token 预算 |
|------|---------|-----------|
| `minimal` | Sonnet 4.6 | 4K |
| `low` | Sonnet 4.6 | 4K |
| `medium` | Sonnet 4.6 | 8K |
| `high` | Sonnet 4.6 | 16K |
| `xhigh` | Opus/Sonnet 4.7+ | 32K |

```bash
hermes model set reasoning.effort medium
```

### 7.3 多模型支持

```
支持提供商（通过 API 或 OpenRouter 聚合）：
  Anthropic / OpenAI / AWS Bedrock / Google Gemini
  Meta Llama / Mistral / xAI Grok / Nous Hermes
  200+ 模型通过 OpenRouter

API 协议：
  - chat_completions（OpenAI 风格）
  - anthropic_messages（Anthropic 原生）
  - bedrock_converse（AWS）

切换命令：
  hermes model set openrouter/meta-llama/llama-3-70b
```

### 7.4 子代理委托（`delegate_tool.py`）

- 创建隔离的 `AIAgent` 子进程
- 独立上下文窗口，不污染父代理
- 支持并行多工作流
- Python 脚本可通过 RPC 调用工具

---

## 8. 架构设计模式总结

| 模式 | 应用位置 | 目的 |
|------|---------|------|
| **适配器模式** | `*_adapter.py` | 统一不同 LLM API 差异 |
| **策略模式** | `terminal_tool.py`, `browser_providers/` | 运行时选择后端实现 |
| **工厂模式** | `model_tools.py`, `platforms/` | 动态创建工具/平台实例 |
| **观察者模式** | `hooks.py`（生命周期钩子） | 插件异步事件通知 |
| **装饰器模式** | `@registry.register()` | 工具零侵入自注册 |
| **模板方法** | `BasePlatformAdapter` | 平台通用流程骨架 |
| **构建器模式** | `prompt_builder.py` | 复杂系统提示组装 |
| **责任链** | 故障转移、重试链 | 错误透明恢复 |
| **单例模式** | `MemoryManager`, `SessionDB` | 全局状态一致 |
| **命令模式** | `cronjob_tools.py`, `todo_tool.py` | 延迟/调度执行 |

---

## 架构总结

Hermes Agent 的力量在于其**可组合性**与**可插拔性**：

- **模型无关** → 切换提供商无需改代码
- **工具模块化** → 新增工具只需 `registry.register()`
- **平台无关** → 一个网关支持 20+ 消息平台
- **可插拔记忆** → 通过 `MemoryProvider` 接口接入任意后端
- **学习闭环** → 技能从经验中创建，记忆跨会话积累
- **企业就绪** → 多租户、审批、PII 移除、成本追踪

核心抽象（`AIAgent` · `BasePlatformAdapter` · `MemoryProvider` · `TerminalEnvironment`）保持了良好的关注点分离，使系统能够在保持一致用户体验的同时持续演进。

---

*文档由代码分析自动生成 · 如有更新请参考最新源码*
