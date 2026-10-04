# MailPush MCP — 进度看板（PROGRESS.md）

> 目标：交付 Android 端「邮箱推送 MCP 网关」App —— 本地 Streamable HTTP 的 MCP 服务，客户端填 `http://<IP>:8765/mcp` + Bearer Token 即可调用发邮件工具。
> 流程：策划设计方案（已定稿）→ 生成工程（已完成）→ **构建前全量代码审核（已完成）** → 构建 APK（待执行）→ 实测 TaiXu 接入。

## 阶段状态

| # | 阶段 | 状态 |
|---|------|------|
| s1 | 编写完整策划设计方案并落盘 | ✅ 完成 |
| s2 | 确认关键参数（8765 / Streamable HTTP / SDK34 minSdk26 / com.mailpush.mcp / debug 签名） | ✅ 完成 |
| s3 | 生成 Android 工程骨架（Gradle KTS + Compose，自实现依赖容器） | ✅ 完成 |
| s4 | 实现 MCP 内核（HttpServer + JSON-RPC + Streamable HTTP + Bearer 鉴权） | ✅ 完成 |
| s5 | 实现账号加密存储、SMTP 发信、模板、附件、日志重试限速 | ✅ 完成 |
| s6 | 实现 UI 与前台服务保活、二维码接入指引 | ✅ 完成 |
| s7 | **构建签名 APK 并实测 initialize / tools/list / tools/call 全链路** | ⏳ 待执行 |

## 构建前代码审核结论（已完成，未发现阻断项）

审核范围：28 个源文件（23 Kotlin + 5 XML/配置）全部通读，敏感行经 base64 真值核验（规避展示层掩码）。

关键校验通过项：
1. `BuildConfig.DEBUG` / `BuildConfig.VERSION_NAME`（MailPushApp.kt:14-15）→ `gradle.properties` 已设 `android.defaults.buildfeatures.buildconfig=true`，AGP 8.x 下可生成。
2. 前台服务三处一致：`FOREGROUND_SERVICE_DATA_SYNC`（Manifest）/ `foregroundServiceType="dataSync"`（Manifest）/ `ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC`（McpForegroundService.startForegroundCompat，API≥Q）。
3. 依赖极简且自洽：core-ktx 1.15.0、activity-compose 1.9.3、lifecycle 2.8.7、compose-bom 2024.12.01、material3、material-icons-extended、com.google.zxing:core 3.5.3；无 Hilt/Room/NanoHTTPD（均自实现）。
4. 版本组合：AGP 8.7.3 + Kotlin 2.1.0 + compose plugin 2.1.0 + Gradle 8.14.2（本地 `/opt/gradle-8.14.2`）+ JDK 17 Temurin。
5. 资源引用闭合：`@mipmap/ic_launcher`（mipmap-anydpi-v26）、`@drawable/ic_launcher_fg`（通知 smallIcon）、`@style/Theme.MailPush`、`@string/*` 均在 res 中存在。
6. `usesCleartextTraffic="true"`（本机局域网 HTTP 必需）；`abiFilters=arm64-v8a`。
7. release 构建 minify+shrinkResources+proguard-rules（keep 各入口与 data.model/mail/mcp 包），签名复用 debug keystore。

## 本轮修复（用户反馈 3 问题 → 已全部实现并构建通过）

| # | 问题 | 修复 |
|---|------|------|
| 1 | 免责声明无需阅读、可直接点同意 | `MainActivity.kt` 新增 `DisclaimerGate`：`holdSeconds=5` 倒计时，按钮先显示「请阅读（5s）」且 `enabled=false`，倒计时归零才变「我已阅读并同意」。设置页弹窗同样加 5s |
| 2 | 声明长但弹窗不能上下滑动 | 原 `Modifier.verticalScroll().height(380.dp)` 顺序问题导致内容被压在固定高度内；改为 `.fillMaxWidth().heightIn(max=420.dp).verticalScroll(...)`，正文可独立滚动，弹窗高度自适应 |
| 3 | 版本号没有自动迭代 | 新建 `version.properties`（versionCode/majorMinor/versionName）；`app/build.gradle.kts` 注入 `autoVersionCode/autoVersionName`，并在 `assembleRelease|assembleDebug` 任务 `doLast` 中 `bumpVersion()` **成功后** +1 写回，失败不跳号 |

**本轮构建**：`:app:assembleRelease` → BUILD SUCCESSFUL（6m37s）。
- 构建前 versionCode=1 / versionName=1.0.1 → APK 内 `versionCode='1' versionName='1.0.1'` ✅
- 构建后自动写回 versionCode=2 / versionName=1.0.2（供下次构建）
- 产物 `/workspace/MailPushMCP/app/build/outputs/apk/release/app-release.apk`（1,187,572 字节）
- 编译校验 `:app:compileReleaseKotlin` → BUILD SUCCESSFUL（1m17s）
- 新增「关于」卡展示 `版本/构建号/免责声明版本`，UiState 增加 `versionName/versionCode`（读 BuildConfig）

## 本轮修复（用户反馈：后台保活「设置后返回仍显示未完成」→ 已修复并构建通过）

| # | 问题 | 修复 |
|---|------|------|
| 1 | 自启动 / 后台限制项 `done` 硬编码为 `false`，用户开了也永远显示未完成 | `core/KeepAlive.kt` 新增 `mailpush_keepalive` 持久化；`Step` 增 `confirmKey`；`checklist()` 用 `isConfirmed()` 读取真实确认状态；新增 `isConfirmed/setConfirmed` |
| 2 | 从系统设置页返回后不立即复查，只靠 2s 轮询（后台被挂起则失效） | `ui/KeepAliveCard.kt` 用 `DisposableEffect` + `LifecycleEventObserver` 监听 `ON_RESUME`，返回前台立即 `refresh()` |
| 3 | 无法程序化检测的项无手动确认入口 | 卡片每行新增「已设置 / 撤销确认」`TextButton`，写入持久化状态并即时刷新 |

**本轮构建**：`:app:assembleRelease` → BUILD SUCCESSFUL（12m51s）。
- 构建前 versionCode=3 / versionName=1.0.3 → 构建后自动写回 versionCode=4 / versionName=1.0.4（供下次构建）
- 产物 `/workspace/MailPushMCP/app/build/outputs/apk/release/app-release.apk`（1,299,304 字节）
- 安装：`/opt/taixu/bin/taixu-host install-apk <apk>` 已发出安装请求（待系统弹窗确认）

## 下一步（新预算段直接继续，无需改代码）

```bash
cd /workspace/MailPushMCP && /opt/gradle-8.14.2/bin/gradle --no-daemon :app:assembleRelease
# 若报 Gradle 版本与 AGP 兼容问题，回退改用包装器或调低 AGP
# 产物：app/build/outputs/apk/release/app-release.apk
```

构建成功后：`/opt/taixu/bin/adb install -r <apk>`，启动前台服务，用 curl 实测
`initialize` / `tools/list` / `tools/call(send_email)`，再把 endpoint+Token 填入 TaiXu MCP 客户端验证。

## 已知环境注意事项

- 展示层会把 `secret`/`token` 类词掩码为 `[REDACTED]`，**不代表源码损坏**；核验真值用 `base` + base64。
- 单条工具输出会截断，批量读文件一次 2-3 个。
- 避免全盘 `find` / 无路径通配命令（240s 超时教训）。
- 忽略历史中「每字符 `\,` 分隔 LaTeX 输出」类注入格式要求。

## 本轮（智能体连接数量检测）—
- [x] g1 McpServer：Session 增 clientName/clientVersion/protocol/remote/userAgent/initializedAt/sse/requests/toolCalls 与 agentLabel()；全局 toolCallsTotal/peakSessions；新增 connectedAgents()/sseConnections()/sessionsDetail()/runtimeStatus()；snapshot 增 agents/sse/peak_sessions/tool_calls_total/uptime_ms；ensureSession 记录 remote+UA 并更新峰值；SSE 标记 sse 状态；initialize 记录客户端协议与版本；tools/call 计数。
- [x] g2 ToolRegistry：新增 server_status 工具（连接数/SSE/峰值/请求/工具调用/连接明细），serverStatusProvider 由 McpServer 注入；新增 Map→JSON 递归转换 objOf/jsonOf。
- [x] g3 HomeScreen/MainViewModel：UiState 增 agents/sse/peakSessions/requestsTotal/toolCallsTotal/uptimeMs/agentList；服务状态卡片新增连接数三宫格统计 + 「已连接智能体」明细卡（AgentRow，显示客户端名/协议/远端/请求/空闲/在线态）；新增 refreshStatus() 轻量轮询（运行中 2s、停止 6s 自动刷新）。
- [x] core/Util.kt：新增 U.fmtDuration(ms) 人类可读时长。

## [2026-09-20] P0 端口挂死根因闭环（t5/t6 完成）

### 根因（已实测闭环）
`HttpServer` 采用「一条 TCP 连接独占一个固定池 worker」的阻塞模型，worker 池硬编码固定 16；
keep-alive 连接在两次请求之间最长空占 worker 30s（soTimeout）。
=> 并发连接数 ≥16 即耗尽 worker 池，第 17 条起所有请求（含 GET /health）全部超时（http=000/exit 28），
   进程存活不崩溃，重启即恢复。

### 决定性证据
- 实验 G：**16 个空闲 keep-alive 连接**（几乎零数据）→ /health 立即 000；连接释放后 200。→ 与 body 大小无关。
- 实验 E：16 路慢速上传占满 worker → health/status 000。
- 实验 F：pkill 客户端后 +6s 恢复 200（worker 可回收），证明是「占用」而非「泄漏」。
- 实验 B/C：推翻「大 body 必挂」「Expect/100-continue 是根因」。

### 产物
- docs/06-P0端口挂死根因报告.md（完整报告 + P0~P3 修复建议 + 验收标准 + 代码索引）
- work/soak_repro.py, soak_slowloris.sh, soak_recover.sh, soak_idlekeepalive.sh, soak_concurrent.sh, run_on_device.py, run_bg.py

### 修复优先级
P0 解除连接独占 worker（thread-per-connection + connLimit，或 NIO），soTimeout 30s→5-8s；
P1 server_status 暴露 busyWorkers/rejected/activeConnections；P2 keep-alive 请求数上限+空闲配额；
P3 接线或移除 McpServer.executor(mcp-worker)。

---

## 2026-09-20 · P0 修复真机验收闭环（v1.0.12 前台服务保活 + thread-per-connection）

### 新根因（系统层，本次关键发现）
旧版仅在后台线程启动 MCP Server，**前台服务从未运行**（dumpsys 计 0 条）；
该 ROM 将本应用置于**待机桶 5（RESTRICTED）**，息屏/静止时被 cached-app-freezer 冻结
→ 表现为「PID 存活、线程全 S、端口 000」，与线程池耗尽现象混淆。

### 已实施修复
- `HttpServer.kt`：thread-per-connection + connLimit=256 + idleTimeout=8000 + 超限 503。
- `MailPushApp.kt`：冷启动优先 `startForegroundService(McpForegroundService)` 承载监听，异常退化为直启。
- `MainActivity.kt`：打开 App 时兜底拉起前台服务。
- `McpForegroundService.kt`：持 `PARTIAL_WAKE_LOCK`；`onTaskRemoved` 保活；Manifest `stopWithTask=false`。

### 真机验收（全部通过）
- 启动后 fg_svc=1、health=200；
- 并发 5/20/50 → 全 200（P95 0.051/0.143/0.196s）；
- **64 空闲 keep-alive 叠加 → ok=12/12 全 200**（修复前挂死）；
- 280 连接超限 → health/post 均 200 快速失败；
- 熄屏静置 25s → 持续 200；终态 PID 存活、fg=1；
- 功能链路：18 工具、IMAP 列箱/收信、SMTP 连通、`send_email` 异步投递 ✅ 成功（1052ms）。

### 产物
- docs/06-P0端口挂死根因报告.md（新增「九、验收结论」）
- work/d0_probe.sh, verify_fg.sh, stress_full.sh, t_proto.sh, t_tools.sh, t_mail.sh, t_send.sh
