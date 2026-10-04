# MailPush MCP

> 把 Android 手机变成一台 **MCP 邮箱网关**：让 AI 助手直接收发你的邮件。

基于 **MCP（Model Context Protocol）** 的 Android 应用，在本机运行一个 HTTP MCP 服务端，
让任意支持 MCP 的 AI 客户端（Claude Desktop / Cursor / 自研智能体等）能够：

- 📮 **发邮件** —— 纯文本 / HTML / 模板 / 批量群发 / 定时发送
- 📥 **收邮件** —— 读收件箱、搜索、提取验证码、按线程找回复
- 🔁 **自动应答** —— 收到新邮件由 AI 自动生成回复（OpenAI 兼容模型）
- 🔐 **安全区** —— 密钥验证识别机主；无密钥的敏感请求自动拦截
- 📊 **统计** —— 今日收件 / 自动回复 / 发信成败一览

无需服务器、无需中转，**邮件账号与凭据全部留在手机上**。

---

## ✨ 特性

### 发信
| 能力 | 说明 |
|---|---|
| `send_email` | 纯文本 / HTML 正文，支持多收件人、抄送密送 |
| `send_template_email` | 使用模板 + `{{变量}}` 占位符替换 |
| `send_batch_email` | 批量群发，逐项可指定不同账号 |
| `schedule_email` | 定时投递 |
| 附件 | 支持本地文件附件 |

**异步发信队列**：调用立即返回 `task_id`（毫秒级），后台单线程串行投递，
`get_send_status` 查询结果。服务器不会被 SMTP 阻塞。

### 收信
| 能力 | 说明 |
|---|---|
| `list_emails` / `read_email` | 收件箱列表与正文读取（IMAP） |
| `search_emails` | 按发件人 / 主题 / 关键词搜索 |
| `get_verification_code` | 自动提取验证码 |
| `list_replies` | **按 In-Reply-To + 主题双策略**找出某封信的所有回复 |
| `list_pending_mails` | 待处理来信队列（持久化，AI 离线也不丢） |

### 自动回复（AI 在环）
```
来信 → 轮询/事件触发发现 → 持久队列（不丢）
     → SSE 推在线 AI（AI 自己回）
     → 内置 AI 模型生成回复（OpenAI 兼容）
     → 兜底回执（只回「已收到」）
     → 本地通知提醒
```
- 支持任意 **OpenAI 兼容端点**（OpenAI / DeepSeek / 通义 / 本地 vLLM）
- 可**自动获取模型列表**，无需手填模型名
- 每日回复数量上限、账号白名单

### 安全区
- 来信带**正确安全密钥** → 识别为机主，无条件配合
- 验证通过后自动加入**可信白名单**（默认 15 天免密钥）
- 无密钥的敏感请求（密码/转账/数据导出等）→ **自动拦截**
- **验证码邮件不自动回复**（只记录通知）
- 引用内容自动剥离，避免原文敏感词误判

### 保活（Android 后台限制对抗）
- 前台服务 + `PARTIAL_WAKE_LOCK`
- **浮窗保活**：悬浮球常驻，提升进程 oom_adj
- 收件箱轮询 + 解锁/网络/回前台事件触发（15 秒节流）

---

## 🚀 快速开始

### 环境要求
- Android 8.0+（minSdk 26）
- JDK 17
- Android SDK（compileSdk 35）

### 构建

```bash
git clone https://github.com/<你的用户名>/MailPushMCP.git
cd MailPushMCP

# 指定 Android SDK 路径
echo "sdk.dir=/path/to/android-sdk" > local.properties

# 编译
./gradlew assembleDebug
```

产物：`app/build/outputs/apk/debug/app-debug.apk`

**正式版**（R8 裁剪，体积约 1.4 MB）：
```bash
./gradlew assembleRelease
```

### 使用

1. 安装 APK，打开 App
2. **账号** 页添加邮箱（SMTP + IMAP，支持常见邮箱预设）
3. **设置 → 服务与网络** 选择监听方式：
   - `仅本机` —— 只绑 `127.0.0.1`，最安全
   - `局域网` —— 绑 `0.0.0.0`，同网段设备可访问
4. 首页可一键复制 **连接地址** 与 **MCP 客户端配置 JSON**
5. 在 AI 客户端中填入地址（+ 令牌）

---

## 🔌 MCP 接入

### 请求头（必须）

```
Content-Type: application/json; charset=utf-8
Accept: application/json, text/event-stream
Authorization: Bearer <访问令牌>
```

> ⚠️ `Accept` 必须同时包含两个值，否则请求会被拒。

### 调用示例

```bash
curl -X POST "http://192.168.1.10:3641/mcp" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "Accept: application/json, text/event-stream" \
  --data-binary @req.json
```

`req.json`（**UTF-8 无 BOM**）：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "send_email",
    "arguments": {
      "to": "someone@example.com",
      "subject": "中文主题",
      "body": "<h2>中文正文</h2>",
      "html": true
    }
  }
}
```

### ⚠️ 中文乱码的三条铁律

1. `Content-Type` **必须带** `charset=utf-8`
2. 用 **`--data-binary @文件`**，绝不用 `-d '中文字符串'`
3. HTML 正文**必须传** `"html": true`

> 服务端编码链已正确处理：主题 RFC 2047、正文 Base64、SMTP 直写 ASCII 字节。
> 乱码几乎都出在调用方的请求头与传参方式。

📖 **完整规范见 [`docs/MCP接入规范-中文与HTML邮件.md`](docs/MCP接入规范-中文与HTML邮件.md)**

---

## 🧰 工具清单（23 个）

| 分类 | 工具 |
|---|---|
| **发信** | `send_email` `send_template_email` `send_batch_email` `schedule_email` `get_send_status` |
| **账号模板** | `list_accounts` `test_account` `list_templates` |
| **收信** | `list_emails` `read_email` `search_emails` `get_verification_code` `list_recent_codes` `mark_email_seen` `list_mailboxes` `test_inbox` |
| **线程待处理** | `list_replies` `list_pending_mails` |
| **安全区** | `list_trusted_senders` `manage_trusted_sender` |
| **记录状态** | `list_email_logs` `server_status` |

---

## 🏗 架构

```
┌─────────────────────────────────────────────────┐
│                   AI 客户端                      │
│         (Claude Desktop / Cursor / 自研)         │
└───────────────────┬─────────────────────────────┘
                    │ MCP over HTTP (JSON-RPC)
                    │
┌───────────────────▼─────────────────────────────┐
│              MailPush MCP (Android)             │
│                                                 │
│  HttpServer ── thread-per-connection            │
│      │         connLimit 256 / idle 8s          │
│      ▼                                          │
│  McpServer ── sessions + SSE 推送               │
│      │                                          │
│      ├─ ToolRegistry（23 个工具）                │
│      │                                          │
│      ├─ MailService ── SMTP / IMAP              │
│      │      │                                   │
│      │      └─ SendQueue（异步投递 + 限速）      │
│      │                                          │
│      ├─ InboxMonitor（轮询 + 事件触发）          │
│      │      └─ AutoReplyEngine                  │
│      │            ├─ SafeReplyGate（安全区）     │
│      │            ├─ TrustedSenderStore（白名单）│
│      │            ├─ AiReplyService（模型调用）  │
│      │            └─ PendingMailStore（持久队列）│
│      │                                          │
│      └─ Store（SQLite v3）                      │
│             accounts / templates                │
│             email_logs / receive_logs           │
└─────────────────────────────────────────────────┘
```

### 关键技术点

**HTTP 服务端**
- `thread-per-connection` 模型（避免长连接独占线程池）
- 连接数上限 256，超限快速返回 503
- 空闲 keep-alive 8 秒回收
- 实测：64 条空闲 keep-alive 并发下服务仍可用

**数据存储**
- 账号凭据用 `EncryptedSharedPreferences` / AES 加密
- 收件日志用于统计，按 uid 唯一索引去重

**后台保活**
- 前台服务（`dataSync` 类型）+ 唤醒锁
- 浮窗保活（`TYPE_APPLICATION_OVERLAY`）
- 应对国产 ROM 的待机桶限制与 cached-app-freezer

---

## 📁 项目结构

```
MailPushMCP/
├── app/src/main/java/com/mailpush/mcp/
│   ├── core/            # 工具类、保活、浮窗
│   ├── data/            # Settings、Store、加密、模型
│   ├── mail/            # SMTP/IMAP、MIME、AI 回复、安全区
│   ├── mcp/             # MCP 服务端、HTTP、工具注册表
│   ├── service/         # 前台服务、广播接收器
│   └── ui/              # Compose 界面
├── docs/                # 设计与规范文档
└── gradle/wrapper/
```

---

## 📄 文档

- [`docs/01-策划设计方案.md`](docs/01-策划设计方案.md) —— 产品与架构设计
- [`docs/06-P0端口挂死根因报告.md`](docs/06-P0端口挂死根因报告.md) —— 连接模型问题排查
- [`docs/MCP接入规范-中文与HTML邮件.md`](docs/MCP接入规范-中文与HTML邮件.md) —— **接入必读**

---

## ⚠️ 免责声明

本项目仅供**个人学习与技术研究**使用。

- 请遵守邮箱服务商的**服务条款**与**发送限制**，勿用于垃圾邮件
- 请妥善保管访问令牌，局域网模式建议开启令牌校验
- 请勿将本工具用于任何违法用途
- 使用本软件产生的一切后果由使用者自行承担

---

## 📜 License

MIT License
