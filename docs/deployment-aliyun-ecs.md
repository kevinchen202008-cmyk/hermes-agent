# Hermes Agent — 阿里云 ECS 部署与运维指南

> **消息平台**：微信（个人号）+ 飞书（企业机器人）  
> **适用版本**：v0.10.0+  
> **最后更新**：2026-04-20

---

## 目录

1. [环境要求](#1-环境要求)
2. [首次安装](#2-首次安装)
3. [配置 LLM 提供商](#3-配置-llm-提供商)
4. [配置飞书机器人](#4-配置飞书机器人)
5. [配置微信个人号](#5-配置微信个人号)
6. [启动与服务管理](#6-启动与服务管理)
7. [阿里云安全组配置](#7-阿里云安全组配置)
8. [验证与测试](#8-验证与测试)
9. [版本升级](#9-版本升级)
10. [备份与恢复](#10-备份与恢复)
11. [常见问题排查](#11-常见问题排查)
12. [运维参考](#12-运维参考)

---

## 1. 环境要求

### 1.1 ECS 最低配置

| 资源 | 最低 | 推荐（双平台网关） |
|------|------|-------------------|
| CPU | 1 核 | 2 核 |
| 内存 | 1 GB | 2 GB |
| 磁盘 | 10 GB | 20 GB |
| 操作系统 | Ubuntu 20.04 LTS | Ubuntu 22.04 LTS |
| 出站网络 | 必须 | 5 Mbps+ |

> **无需 GPU**：Hermes 所有 LLM 调用均走 API，计算在云端，本机仅作网关/协调层。

### 1.2 网络出站要求

| 目标域名 | 用途 | 是否必须 |
|----------|------|---------|
| `github.com` | 代码更新 | 安装 / 升级时必须 |
| `openrouter.ai` / `api.anthropic.com` | LLM API | 必须（任选其一） |
| `open.feishu.cn` | 飞书 WebSocket | 飞书平台必须 |
| `ilinkai.weixin.qq.com` | 微信 iLink 服务 | 微信平台必须 |
| `novac2c.cdn.weixin.qq.com` | 微信 CDN | 微信媒体必须 |

> **大陆 ECS 注意**：访问 `github.com` 和 `api.anthropic.com` 可能需要配置出口代理。

### 1.3 前提软件

安装脚本会自动处理 Python 3.11 和 Node.js 22 的安装，但需要系统预装以下工具：

```bash
sudo apt-get update && sudo apt-get install -y \
    git curl wget \
    build-essential python3-dev libffi-dev \
    ripgrep ffmpeg
```

---

## 2. 首次安装

### 2.1 一键安装（推荐）

SSH 登录 ECS 后执行：

```bash
# 若需通过代理访问 GitHub，先设置代理
# export HTTPS_PROXY=http://your-proxy:port

curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh \
  | bash -s -- --skip-setup
```

脚本自动完成：

| 步骤 | 说明 |
|------|------|
| 安装 `uv` | 高速 Python 包管理器 |
| 检测/安装 Python 3.11 | 若系统版本不足则自动安装 |
| 安装 Node.js 22 | 用于浏览器自动化（Playwright） |
| Clone 代码 | 到 `~/.hermes/hermes-agent/` |
| 创建虚拟环境 | `~/.hermes/hermes-agent/venv/` |
| 安装 Python 依赖 | `.[all]` 全功能包 |
| 初始化目录结构 | `~/.hermes/` 各子目录 |
| 注册 `hermes` 命令 | `~/.local/bin/hermes` |

安装完成后重载 Shell：

```bash
source ~/.bashrc   # 或 source ~/.zshrc
```

### 2.2 手工安装（可选，适合自定义场景）

```bash
# 1. 安装 uv
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"

# 2. 克隆代码
HERMES_HOME="${HOME}/.hermes"
INSTALL_DIR="${HERMES_HOME}/hermes-agent"
git clone https://github.com/NousResearch/hermes-agent.git "${INSTALL_DIR}"
cd "${INSTALL_DIR}"

# 3. 创建虚拟环境并安装依赖
uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all]"

# 4. 注册命令
mkdir -p ~/.local/bin
ln -sf "${INSTALL_DIR}/venv/bin/hermes" ~/.local/bin/hermes

# 5. 初始化配置目录
mkdir -p "${HERMES_HOME}"/{cron,sessions,logs,hooks,memories,skills,workspace,home}
cp "${INSTALL_DIR}/.env.example"              "${HERMES_HOME}/.env"
cp "${INSTALL_DIR}/cli-config.yaml.example"   "${HERMES_HOME}/config.yaml"

# 6. 同步内置技能
python "${INSTALL_DIR}/tools/skills_sync.py"
```

### 2.3 安装后目录结构

```
~/.hermes/
├── .env                    # API 密钥与平台凭证（敏感，chmod 600）
├── config.yaml             # 运行配置
├── SOUL.md                 # Agent 个性定义（可自定义）
├── MEMORY.md               # 持久化用户记忆
│
├── hermes-agent/           # 代码库（git 仓库）
│   └── venv/               # Python 虚拟环境
│
├── skills/                 # 内置 + 用户技能
├── sessions/               # 会话路由索引
├── cron/                   # 定时任务定义与日志
│   ├── jobs.json
│   └── runs/
├── logs/                   # 应用日志（含 update.log）
├── memories/               # 插件记忆存储
├── workspace/              # Agent 工作目录
└── home/                   # 子进程 HOME（git/ssh/npm 配置）
```

### 2.4 验证安装

```bash
hermes --version    # 应显示 v0.10.0 或更新版本
hermes doctor       # 自动诊断所有依赖和配置
```

---

## 3. 配置 LLM 提供商

编辑 `~/.hermes/.env`，填入至少一个 LLM 提供商的密钥：

```bash
nano ~/.hermes/.env
```

### 3.1 推荐：OpenRouter（支持 200+ 模型）

```bash
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

配置 `~/.hermes/config.yaml`：

```yaml
model:
  provider: openrouter
  default: anthropic/claude-sonnet-4-6   # 或其他模型
```

### 3.2 直连 Anthropic

```bash
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 3.3 大陆 ECS 代理配置

若 ECS 无法直连 LLM API，在 `.env` 中加入：

```bash
HTTPS_PROXY=http://your-proxy-host:port
HTTP_PROXY=http://your-proxy-host:port
NO_PROXY=localhost,127.0.0.1,open.feishu.cn
```

保护密钥文件权限：

```bash
chmod 600 ~/.hermes/.env
```

---

## 4. 配置飞书机器人

飞书使用 **WebSocket 长连接**模式，无需开放任何入站端口。

### 4.1 在飞书开放平台创建应用

1. 打开 https://open.feishu.cn（国内）或 https://open.larksuite.com（国际版 Lark）
2. 登录企业管理员账号
3. 点击「创建应用」→ 选择「**企业自建应用**」（无需审核，立即生效）
4. 填写应用名称（如：`Hermes Agent`）和描述
5. 进入应用管理页，左侧选「**添加应用能力**」→ 开启「**机器人**」
6. 左侧选「**权限管理**」→ 开通以下权限：

| 权限标识 | 说明 |
|---------|------|
| `im:message` | 接收和发送消息 |
| `im:message.group_at_msg` | 接收群聊 @ 消息 |
| `im:message.group_at_msg:readonly` | 读取群聊消息 |
| `im:resource` | 发送图片/文件 |

7. 左侧「**应用发布**」→ 点击「**创建版本**」→ 「**申请线上发布**」（企业内部应用直接通过）
8. 回到「**凭证与基础信息**」页，记录：
   - **App ID**（格式：`cli_xxxxxxxxxxxxxxxxx`）
   - **App Secret**

### 4.2 在 Hermes 中配置

**方式 A：交互向导（推荐）**

```bash
hermes gateway setup
# 从列表中选择 "Feishu / Lark"
# → 选择手动输入凭证
# → 输入 App ID
# → 输入 App Secret
# → 选择域：feishu（国内）/ lark（国际）
# → 连接模式：选 WebSocket（推荐）
```

**方式 B：直接编辑 .env**

```bash
# 追加到 ~/.hermes/.env
FEISHU_APP_ID=cli_xxxxxxxxxxxxxxxxx
FEISHU_APP_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
FEISHU_DOMAIN=feishu                   # 国内用 feishu，国际用 lark
FEISHU_CONNECTION_MODE=websocket       # 无需公网端口
```

### 4.3 群聊接入（可选）

将 Bot 添加到飞书群：

1. 在飞书群聊中，点击右上角「设置」→「群机器人」→「添加机器人」
2. 搜索你的应用名称并添加
3. 之后在群里 @ 机器人即可触发对话

---

## 5. 配置微信个人号

微信接入使用 **Tencent iLink** 协议，需要用**手机微信**扫码授权，为**个人账号**接入（非公众号、非企业微信）。

> **注意**：iLink Token 有时效性，过期后需要重新扫码。建议使用**常用主微信号**或专用的**小号**。

### 5.1 确认依赖

```bash
# 检查 aiohttp 和 cryptography 是否已安装（[all] 安装后应已包含）
python -c "import aiohttp, cryptography; print('OK')"
```

若缺失：

```bash
cd ~/.hermes/hermes-agent
source venv/bin/activate
uv pip install aiohttp cryptography
```

### 5.2 扫码登录

**必须在 SSH 终端交互模式下执行（不能在后台）：**

```bash
hermes gateway setup
# 从列表中选择 "Weixin / WeChat"
# 终端会显示 ASCII QR 码
# 用手机微信扫码 → 点击「确认登录」
```

登录成功后，凭证自动写入 `~/.hermes/.env`：

```bash
WEIXIN_ACCOUNT_ID=xxxxxxxx
WEIXIN_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxx
WEIXIN_BASE_URL=https://...（自动填充）
WEIXIN_CDN_BASE_URL=https://novac2c.cdn.weixin.qq.com/c2c
```

### 5.3 访问控制配置

向导会询问以下策略（也可事后在 `.env` 中修改）：

**私聊策略（推荐配置）：**

```bash
# 推荐：陌生人需审批（安全）
WEIXIN_DM_POLICY=pairing
WEIXIN_ALLOW_ALL_USERS=false
WEIXIN_ALLOWED_USERS=

# 或：仅允许指定用户（最严格）
WEIXIN_DM_POLICY=allowlist
WEIXIN_ALLOW_ALL_USERS=false
WEIXIN_ALLOWED_USERS=wxid_xxxxxx,wxid_yyyyyy   # 逗号分隔的微信 ID
```

**群聊策略（初期建议禁用）：**

```bash
WEIXIN_GROUP_POLICY=disabled   # 禁用（初期推荐）
# 或
WEIXIN_GROUP_POLICY=open       # 全部允许
# 或
WEIXIN_GROUP_POLICY=allowlist  # 白名单
WEIXIN_GROUP_ALLOWED_USERS=group_id_1,group_id_2
```

**Home 频道（Cron 任务投递目标）：**

```bash
WEIXIN_HOME_CHANNEL=wxid_xxxxxx    # 你的微信 ID（setup 向导可自动填充）
```

### 5.4 微信 Token 续期

iLink Token 会在数天到数周后过期，过期后网关日志会出现 `401` 或连接断开。

**手动续期：**

```bash
# 停止网关
systemctl --user stop hermes-gateway

# 重新扫码
hermes gateway setup
# 选 Weixin / WeChat → 选择重新配置 → 扫码

# 重启网关
systemctl --user start hermes-gateway
```

**自动监控（可选）**：设置一个 Hermes Cron 任务，定期检查微信连接状态并通过飞书发送提醒（见 [12. 运维参考](#12-运维参考)）。

---

## 6. 启动与服务管理

### 6.1 安装 Systemd 服务

```bash
hermes gateway install
```

该命令自动创建 `~/.config/systemd/user/hermes-gateway.service` 并启动服务。

### 6.2 服务管理命令

```bash
# 查看状态
systemctl --user status hermes-gateway

# 启动 / 停止 / 重启
systemctl --user start   hermes-gateway
systemctl --user stop    hermes-gateway
systemctl --user restart hermes-gateway

# 开机自启
systemctl --user enable  hermes-gateway
systemctl --user disable hermes-gateway

# 实时日志
journalctl --user -u hermes-gateway -f

# 查看最近 100 行日志
journalctl --user -u hermes-gateway -n 100
```

### 6.3 Systemd 服务文件内容参考

若需手动创建或修改，服务文件位于：

```
~/.config/systemd/user/hermes-gateway.service
```

标准内容示例：

```ini
[Unit]
Description=Hermes Agent Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/home/YOUR_USER/.local/bin/hermes gateway
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=hermes-gateway

Environment=PATH=/home/YOUR_USER/.local/bin:/usr/local/bin:/usr/bin:/bin
Environment=HOME=/home/YOUR_USER
Environment=HERMES_HOME=/home/YOUR_USER/.hermes

[Install]
WantedBy=default.target
```

修改后重载：

```bash
systemctl --user daemon-reload
systemctl --user restart hermes-gateway
```

### 6.4 确保 SSH 退出后服务持续运行

用户级 systemd 服务默认在用户退出 SSH 后停止。执行以下命令使其在后台持续运行：

```bash
# 允许用户服务在未登录时也保持运行
sudo loginctl enable-linger $USER
```

---

## 7. 阿里云安全组配置

飞书 WebSocket 和微信 iLink 均为**出站**连接，无需开放任何入站端口。

| 规则 | 方向 | 端口 | 说明 |
|------|------|------|------|
| SSH 管理 | 入站 | 22/TCP | 必须保留 |
| 飞书 WebSocket | 出站 | 443/TCP | 无需入站规则 |
| 微信 iLink | 出站 | 443/TCP | 无需入站规则 |
| 飞书 Webhook（仅 Webhook 模式） | 入站 | 8765/TCP | WebSocket 模式无需此项 |
| REST API 服务器（可选） | 入站 | 8000/TCP | 仅启用 api_server 平台时需要 |

---

## 8. 验证与测试

### 8.1 查看网关状态

```bash
systemctl --user status hermes-gateway
# 应显示 Active: active (running)

journalctl --user -u hermes-gateway -n 50
# 应看到 "Feishu: connected" 和 "Weixin: connected" 类似信息
```

### 8.2 飞书测试

1. 在飞书中找到你创建的机器人
2. 发送一条消息：`你好`
3. 机器人应在数秒内回复

### 8.3 微信测试

1. 用另一个微信账号（或手机）向绑定账号发送私信：`你好`
2. 若策略为 `pairing`，需先通过审批（运行 `hermes pairing approve`）
3. 审批后发送消息应收到 AI 回复

### 8.4 全面健康检查

```bash
hermes doctor
```

输出示例：
```
✓ Python 3.11.x
✓ Node.js v22.x
✓ hermes command found
✓ OPENROUTER_API_KEY configured
✓ FEISHU_APP_ID configured
✓ WEIXIN_ACCOUNT_ID configured
✓ Gateway service: active
```

---

## 9. 版本升级

### 9.1 标准升级流程

Hermes 内置完整的升级机制，**一条命令**完成所有步骤：

```bash
hermes update
```

`hermes update` 自动执行：

| 步骤 | 说明 |
|------|------|
| `git stash` | 保存本地未提交改动（若有） |
| `git fetch origin` + `git pull --ff-only origin main` | 拉取最新代码 |
| 若历史分叉 | 自动 `git reset --hard origin/main` |
| `uv pip install -e ".[all]"` | 更新 Python 依赖 |
| 清理 `__pycache__` | 防止字节码缓存导致的 ImportError |
| Skills 同步 | 新增/更新内置技能，保留用户自定义 |
| 配置迁移检查 | 提示新增的配置项或环境变量 |
| `git stash pop` | 还原本地改动（若有） |
| 重启 Gateway | 若服务正在运行则自动重启 |

升级日志保存于：`~/.hermes/logs/update.log`

### 9.2 在 SSH 会话中安全升级

`hermes update` 已内置 SSH 断连保护（SIGHUP 忽略 + 输出镜像到日志），可直接在普通 SSH 终端执行，**无需 tmux/screen**。

若仍希望使用 tmux 保险：

```bash
tmux new-session -s update
hermes update
# Ctrl+B, D 脱离会话（升级在后台继续）
```

### 9.3 网关在线升级（不中断服务）

若网关正在运行，可通过消息平台发送斜杠命令触发升级（无需 SSH）：

```
/update
```

等价于在后台运行 `hermes update --gateway`，升级完成后网关自动重启。

### 9.4 Fork 仓库升级说明

本项目 `origin` 指向你的 Fork（`kevinchen202008-cmyk/hermes-agent`），`upstream` 指向主仓（`NousResearch/hermes-agent`）。

`hermes update` 会自动检测 Fork 情况并从 `upstream` 拉取更新。若需手动同步：

```bash
cd ~/.hermes/hermes-agent
git fetch upstream
git merge upstream/main
# 同步到你的 Fork
git push origin main
```

### 9.5 升级后验证

```bash
hermes --version         # 确认版本号已更新
hermes doctor            # 检查所有组件状态
systemctl --user status hermes-gateway   # 确认服务正常
journalctl --user -u hermes-gateway -n 30   # 查看启动日志
```

### 9.6 回滚（升级出问题时）

```bash
# 停止网关
systemctl --user stop hermes-gateway

# 回滚到上一个稳定 commit
cd ~/.hermes/hermes-agent
git log --oneline -10          # 找到想回滚的版本
git checkout <commit-hash>     # 或 git checkout v2026.4.16（使用 tag）

# 重新安装该版本的依赖
source venv/bin/activate
uv pip install -e ".[all]"

# 重启
systemctl --user start hermes-gateway
```

---

## 10. 备份与恢复

### 10.1 创建备份

```bash
# 默认备份到 ~/hermes-backup-YYYY-MM-DD-HHMMSS.zip
hermes backup

# 指定输出路径
hermes backup --output /opt/backups/

# 快速快照（仅核心配置和数据库，不含媒体缓存）
hermes backup --quick
```

备份包含：`.env`、`config.yaml`、`state.db`（会话历史）、`skills/`、`cron/jobs.json`、`MEMORY.md` 等。

### 10.2 定期自动备份

通过 Hermes Cron 设置每日备份：

```bash
hermes cron add \
  --schedule "0 3 * * *" \
  --prompt "Run: hermes backup --output /opt/hermes-backups/ && echo Done" \
  --id daily-backup
```

或使用系统 crontab：

```bash
crontab -e
# 加入：
0 3 * * * /home/YOUR_USER/.local/bin/hermes backup --output /opt/hermes-backups/ >> /home/YOUR_USER/.hermes/logs/backup.log 2>&1
```

### 10.3 从备份恢复

```bash
# 停止网关
systemctl --user stop hermes-gateway

# 恢复备份（会覆盖现有文件）
hermes import /path/to/hermes-backup-2026-04-20-030000.zip

# 重启
systemctl --user start hermes-gateway
```

### 10.4 升级前手动备份（强烈建议）

```bash
hermes backup --output ~/pre-upgrade-backup/
hermes update
```

---

## 11. 常见问题排查

### 11.1 安装类问题

| 现象 | 原因 | 解决方案 |
|------|------|---------|
| `hermes: command not found` | PATH 未更新 | `source ~/.bashrc` 或重新登录 |
| `ModuleNotFoundError` | 虚拟环境未激活 | `source ~/.hermes/hermes-agent/venv/bin/activate` |
| `git clone` 超时/失败 | 大陆访问 GitHub 慢 | 设置 `HTTPS_PROXY` 后重试 |
| `build-essential` 缺失 | 编译依赖不全 | `sudo apt-get install build-essential python3-dev` |
| `requires-python >= 3.11` | Python 版本过低 | `uv python install 3.11` |

### 11.2 飞书连接问题

| 现象 | 原因 | 解决方案 |
|------|------|---------|
| 发消息无响应 | Bot 能力未开启 | 飞书开放平台 → 应用能力 → 开启「机器人」 |
| 权限错误日志 | 权限未申请 | 飞书开放平台 → 权限管理 → 申请所需权限 |
| WebSocket 断连 | 网络抖动 | 网关自动重连，查看日志确认 |
| `App ID not found` | App ID 填错 | 核对 `.env` 中 `FEISHU_APP_ID` 格式（`cli_` 开头）|
| 群聊无响应 | Bot 未加入群 | 在群里「添加机器人」后 @ Bot 才有效 |

### 11.3 微信连接问题

| 现象 | 原因 | 解决方案 |
|------|------|---------|
| QR 码扫描后无反应 | iLink 连接不稳定 | 重试 `hermes gateway setup` → WeChat |
| `401 Unauthorized` | Token 过期 | 重新执行 `hermes gateway setup` 扫码续期 |
| 消息收不到回复 | DM 策略为 pairing | 运行 `hermes pairing approve` 审批用户 |
| `aiohttp not found` | 依赖缺失 | `uv pip install aiohttp cryptography` |
| CDN 图片无法发送 | CDN 域名访问受限 | 检查出站是否允许 `*.cdn.weixin.qq.com` |

### 11.4 网关启动失败

```bash
# 查看详细错误
journalctl --user -u hermes-gateway -n 100 --no-pager

# 直接前台运行调试（输出实时日志）
hermes gateway

# 诊断所有依赖
hermes doctor
```

### 11.5 升级失败

```bash
# 查看升级日志
cat ~/.hermes/logs/update.log

# 手动重新安装依赖
cd ~/.hermes/hermes-agent
source venv/bin/activate
uv pip install -e ".[all]"

# 若 git 状态异常
git status
git stash list    # 查看是否有 stash 未还原
git stash pop     # 还原（如需要）
```

---

## 12. 运维参考

### 12.1 常用命令速查

```bash
# 启动交互式 CLI 对话
hermes

# 查看/切换模型
hermes model

# 启用/禁用工具集
hermes tools

# 查看网关平台状态
hermes gateway status

# 重新配置消息平台
hermes gateway setup

# 查看/管理定时任务
hermes cron list
hermes cron add
hermes cron remove <id>

# 配置项操作
hermes config set model.default anthropic/claude-opus-4-6
hermes config get model.default

# 配置迁移（升级后如有新配置项）
hermes config migrate

# 诊断
hermes doctor

# 升级
hermes update

# 备份
hermes backup
```

### 12.2 监控微信连接状态（可选 Cron）

在飞书中添加一个定时检查任务，当微信断连时通过飞书提醒：

```bash
hermes cron add \
  --schedule "0 */6 * * *" \
  --prompt "检查微信连接状态，若日志中有断连迹象请通过飞书提醒我" \
  --id check-weixin \
  --deliver feishu
```

### 12.3 日志位置

| 日志 | 路径 |
|------|------|
| 应用主日志 | `~/.hermes/logs/` |
| 升级日志 | `~/.hermes/logs/update.log` |
| 备份日志 | `~/.hermes/logs/backup.log`（自定义） |
| Systemd 日志 | `journalctl --user -u hermes-gateway` |
| Cron 执行日志 | `~/.hermes/cron/runs/` |

### 12.4 版本信息查看

```bash
hermes --version
# 或
cat ~/.hermes/hermes-agent/hermes_cli/__init__.py | grep __version__

# 查看最新提交
cd ~/.hermes/hermes-agent && git log --oneline -5
```

### 12.5 多配置文件（Profile）

若需在同一 ECS 上运行多个独立 Agent 实例（如个人用 + 团队用）：

```bash
# 创建新 profile
hermes profile create work

# 切换 profile
hermes profile use work

# 当前 profile 独立存储：
# ~/.hermes/profiles/work/{config.yaml, state.db, skills/, ...}
```

---

## 附录：关键文件与目录速查

| 文件 / 目录 | 用途 |
|------------|------|
| `~/.hermes/.env` | API 密钥、平台凭证（chmod 600） |
| `~/.hermes/config.yaml` | 模型、工具、网关配置 |
| `~/.hermes/SOUL.md` | Agent 个性/角色定义 |
| `~/.hermes/MEMORY.md` | 跨会话持久化记忆 |
| `~/.hermes/state.db` | SQLite 会话数据库 |
| `~/.hermes/skills/` | 内置 + 用户自定义技能 |
| `~/.hermes/cron/jobs.json` | 定时任务定义 |
| `~/.hermes/logs/update.log` | 升级日志 |
| `~/.hermes/hermes-agent/` | 代码库（git 仓库） |
| `~/.hermes/hermes-agent/venv/` | Python 虚拟环境 |
| `~/.config/systemd/user/hermes-gateway.service` | Systemd 服务文件 |

---

*文档版本：v1.0 · 适用于 Hermes Agent v0.10.0+*
