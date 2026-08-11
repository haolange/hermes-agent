---
kind: logging_system
name: Hermes Agent 集中式日志系统：多文件轮转、组件路由与敏感信息脱敏
category: logging_system
scope:
    - '**'
source_files:
    - hermes_logging.py
    - agent/redact.py
    - hermes_cli/logs.py
    - hermes_cli/main.py
    - agent/agent_init.py
    - acp_adapter/entry.py
---

## 1. 使用的系统与框架

Hermes Agent 使用 Python 标准库 `logging` 作为底层框架，并在根目录的 `hermes_logging.py` 中提供统一的 `setup_logging()` 入口。所有子进程（CLI、Gateway、TUI、Cron、MCP 服务器等）在启动早期调用该函数完成日志初始化。

- **日志存储**：基于 `RotatingFileHandler`（Windows 下通过 `concurrent_log_handler.ConcurrentRotatingFileHandler` 别名替换），按大小轮转并保留若干备份。
- **异步落盘**：所有文件 handler 不直接挂到 root logger，而是通过一个进程内 `queue.SimpleQueue` + `logging.handlers.QueueListener` 在独立线程上写入，避免事件循环被跨进程旋转锁阻塞。
- **脱敏格式化器**：`agent.redact.RedactingFormatter` 继承自 `logging.Formatter`，在 `format()` 阶段对输出文本执行正则脱敏后再写出。
- **会话上下文注入**：通过 `logging.setLogRecordFactory` 包装全局记录工厂，在每个 `LogRecord` 上注入 `session_tag`（形如 `[session_id]`），由 `_install_session_record_factory()` 在模块导入时即安装。

## 2. 关键文件与包

| 文件 | 职责 |
|---|---|
| `hermes_logging.py` | 统一入口 `setup_logging()`、`setup_verbose_logging()`、`set_session_context()`/`clear_session_context()`、`COMPONENT_PREFIXES`、异步队列监听器、`_ManagedRotatingFileHandler` |
| `agent/redact.py` | `RedactingFormatter` 及 `redact_sensitive_text()` / `redact_terminal_output()` / `mask_secret()` 等脱敏规则 |
| `hermes_cli/logs.py` | `hermes logs` 子命令：按 level/session/component/since 过滤查看 `~/.hermes/logs/*.log` |
| `hermes_cli/main.py` | CLI 启动路径中调用 `setup_logging(mode="cli")` 或 `mode="gui"` |
| `agent/agent_init.py` | Agent 运行时调用 `setup_logging(hermes_home=...)` |
| `acp_adapter/entry.py` | ACP 适配器进程也调用自己的 `_setup_logging()` |
| `gateway/run.py` | Gateway 模式（见 grep 结果） |

## 3. 架构与约定

### 3.1 日志文件布局
所有日志位于 `~/.hermes/logs/`（通过 `get_hermes_home()` 解析，支持 profile-aware）。默认产生：
- `agent.log` — INFO+，主活动日志（catch-all）
- `errors.log` — WARNING+，仅错误与告警，便于快速排查
- `gateway.log` — 仅在 `mode="gateway"` 时创建，仅接收 `gateway.*`、`hermes_plugins`、`plugins.platforms` 等前缀的记录
- `gui.log` — 仅在 `mode="gui"` 时创建，仅接收 `hermes_cli.web_server`、`hermes_cli.pty_bridge`、`tui_gateway.*`、`uvicorn.*` 等前缀的记录

### 3.2 组件路由
通过 `COMPONENT_PREFIXES` 字典定义每个组件的 logger name 前缀集合，配合 `_ComponentFilter`（`record.name.startswith(prefixes)`）将消息路由到对应文件。新增组件只需在此字典中添加前缀即可。

### 3.3 级别策略
- 默认 root logger 级别为 `INFO`，可从 `config.yaml` 的 `logging.level` 覆盖。
- `errors.log` 固定为 `WARNING`。
- 第三方噪声日志（`openai`、`httpx`、`grpc`、`websockets`、`urllib3` 等）强制设为 `WARNING`。
- `setup_verbose_logging()` 添加一个 DEBUG 级别的 `StreamHandler` 到 stderr，用于 `--verbose` 模式。

### 3.4 会话关联
通过 `set_session_context(session_id)` / `clear_session_context()` 设置当前线程的会话 ID，所有后续 log 行自动带上 `[session_id]` 标记，便于过滤与关联。

### 3.5 Windows 兼容与外部轮转
- Windows 下用 `concurrent-log-handler` 替代 stdlib `RotatingFileHandler`，解决多进程同时持有句柄导致的 `PermissionError [WinError 32]`；其锁超时异常被 `handleError` 静默吞掉，避免 Desktop slash-worker 把 traceback 发到聊天输出。
- `_ManagedRotatingFileHandler` 在 managed mode（NixOS setgid 目录）下每次 open/rollover 后 `chmod 0660`，保证组内共享可读；同时每 emit 前 stat baseFilename 检测外部轮转（logrotate 等），inode 变化则重新打开流。

### 3.6 异步队列与优雅关闭
所有文件 handler 经 `_register_queued_handler()` 注册到共享 `QueueListener`，emit 线程永不阻塞。提供 `flush_log_queue()`（测试用，阻塞直到队列清空）和 `drain_log_queue(timeout)`（硬退出路径，带超时防止 listener 卡在旋转锁上）。

## 4. 脱敏与安全约束

`agent/redact.py` 维护一套正则驱动的脱敏规则，在 `RedactingFormatter.format()` 中对每条日志输出执行：

- **已知密钥前缀**：`sk-*`、`ghp_*`、`AKIA*`、`xoxb-*`、`AIza*`、`sk_live_*` 等数十种供应商 token 前缀，短于 18 字符的全量掩码，长 token 保留前 6 后 4 位。
- **环境变量赋值**：`KEY=value` 形式中 key 含 `API_KEY`、`TOKEN`、`SECRET`、`PASSWORD` 等关键字时掩码值。
- **JSON/YAML 配置键**：`"apiKey": "value"`、`password: secret` 等命名空间化键。
- **Authorization 头**：`Authorization: Bearer ...` 及 `x-api-key` 等自定义头。
- **数据库连接串**：`postgres://user:PASS@host` 中的密码。
- **JWT**：`eyJ...` 三段式 JWT。
- **Telegram bot token**、**E.164 手机号**、**私钥块**、**URL userinfo 裸 token** 等。
- **Web URL 查询参数默认不过滤**（OAuth 回调、magic-link、预签名 URL 需透传），但可通过 `redact_url_credentials=True` 严格模式开启。
- 全局开关 `HERMES_REDACT_SECRETS`（默认 `true`），可通过 `security.redact_secrets: false` 在 config 中关闭，但关闭时会记录警告。

终端输出走 `redact_terminal_output(output, command)`，根据命令类型（env dump vs file read vs 其他）选择是否运行 ENV 赋值脱敏路径，避免误伤源码/配置片段。

## 5. 调用点与生命周期

- `hermes_cli/main.py`：CLI 入口调用 `setup_logging(mode="cli")`；GUI/TUI 相关路径调用 `mode="gui"`。
- `agent/agent_init.py`：Agent 运行时调用 `setup_logging(hermes_home=...)`。
- `acp_adapter/entry.py`：ACP 适配器进程有自己的 `_setup_logging()`。
- `gateway/run.py`：Gateway 模式（grep 确认存在调用）。
- 各模块通过 `from hermes_logging import set_session_context` 在对话开始时注入 session id。

## 6. 运维与可观测性

- `hermes logs` 子命令支持 `--level`、`--session`、`--component`、`--since`、`--follow` 等过滤选项，复用 `COMPONENT_PREFIXES`。
- 日志目录结构清晰：`~/.hermes/logs/agent.log`、`errors.log`、`gateway.log`、`gui.log`，便于不同角色（平台运维、agent 开发者、dashboard 用户）各取所需。
- 所有日志 UTF-8 编码，stderr 通过 `_safe_stderr()` 包装处理非 UTF-8 控制台编码，避免 UnicodeEncodeError 导致进程崩溃。
