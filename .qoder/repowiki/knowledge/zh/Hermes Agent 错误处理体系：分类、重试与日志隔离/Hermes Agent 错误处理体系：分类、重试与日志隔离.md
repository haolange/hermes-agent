---
kind: error_handling
name: Hermes Agent 错误处理体系：分类、重试与日志隔离
category: error_handling
scope:
    - '**'
source_files:
    - agent/error_classifier.py
    - agent/errors.py
    - gateway/platforms/base.py
    - hermes_logging.py
    - hermes_cli/middleware.py
    - agent/transports/codex_app_server.py
    - agent/lsp/protocol.py
    - agent/pet/store.py
    - agent/pet/manifest.py
    - agent/pet/generate/imagegen.py
    - agent/plugin_llm.py
    - agent/secret_scope.py
    - agent/subagent_lifecycle.py
    - agent/trace_upload.py
    - agent/conversation_compression.py
    - cron/scheduler.py
    - gateway/platforms/api_server.py
    - gateway/platforms/qqbot/chunked_upload.py
    - gateway/platforms/signal_rate_limit.py
    - gateway/turn_lease.py
---

## 1. 总体方法

Hermes Agent 没有使用统一的异常基类或全局中间件框架，而是采用**分层 + 领域化**的错误处理策略：

- **领域专用异常**：每个子系统（agent、gateway/platforms、cron、tools）定义自己的 `*Error` / `*Exception` 子类，通常继承自 `RuntimeError`、`ValueError`、`PermissionError` 等内置类型，便于按类型捕获。
- **API 错误分类器**：`agent/error_classifier.py` 集中维护一个优先级排序的匹配流水线，把任意上游 API 异常归类为 `FailoverReason` 枚举值（auth、rate_limit、billing、context_overflow、ssl_cert_verification、content_policy_blocked、timeout、unknown …），并附带 `retryable`、`should_compress`、`should_rotate_credential`、`should_fallback` 等恢复提示，供上层重试/回退循环消费。
- **网关致命错误状态机**：`gateway/platforms/base.py` 的 `BasePlatformAdapter` 通过 `_set_fatal_error(code, message, retryable)` 将平台适配器置入 fatal 状态，并通过可选的 `set_fatal_error_handler` 回调通知上层进行重连/告警。
- **日志即错误通道**：`hermes_logging.py` 提供统一入口 `setup_logging()`，写入 `agent.log`（INFO+）、`errors.log`（WARNING+ 快速排查）、`gateway.log`（仅 gateway.* 组件）、`gui.log`（dashboard/TUI-gateway）。所有文件经 `RotatingFileHandler`（Windows 下用 `concurrent_log_handler.ConcurrentRotatingFileHandler`）和 `RedactingFormatter` 输出，避免敏感信息落盘。
- **中间件错误传播**：`hermes_cli/middleware.py` 定义了 LLM/tool 请求与执行两类中间件管线。执行链内部用 `_DownstreamExecutionError` 包装下游异常，并在回调自身抛错时记录 warning 并尝试继续后续帧；多次调用 `next_call()` 会抛出 `RuntimeError` 作为契约违规信号。

## 2. 关键文件与包

| 文件 | 职责 |
|---|---|
| `agent/errors.py` | 领域内轻量异常：`SSLConfigurationError`、`EmptyStreamError`、`MoAPresetNotFoundError` |
| `agent/error_classifier.py` | 全仓库最核心的 API 错误分类器，维护数百条 provider 特定模式串，返回 `ClassifiedError` |
| `gateway/platforms/base.py` | 平台适配器的致命错误状态机（`_set_fatal_error`、`fatal_error_code`、`fatal_error_retryable`） |
| `hermes_logging.py` | 统一日志初始化、多文件路由、异步队列监听、跨进程旋转锁保护 |
| `hermes_cli/middleware.py` | 中间件契约与执行链，封装 request/execution 两类钩子及错误传播 |
| `agent/transports/*.py` | 各模型传输层抛出 SDK 原生异常，由分类器消化 |
| `agent/lsp/protocol.py` | LSP 协议层 `LSPProtocolError`、`LSPRequestError` |
| `agent/pet/store.py`、`agent/pet/manifest.py`、`agent/pet/generate/imagegen.py` | Pet 子系统各自 `PetStoreError`、`ManifestError`、`GenerationError` |
| `agent/plugin_llm.py` | `PluginLlmTrustError(PermissionError)` 用于插件信任拒绝 |
| `agent/secret_scope.py` | `UnscopedSecretError(RuntimeError)` 越界访问密钥 |
| `agent/subagent_lifecycle.py` | `SubagentLifecycleError(ValueError)` |
| `agent/trace_upload.py` | `TraceRedactionError(RuntimeError)` |
| `agent/conversation_compression.py` | `CompressionExecutorSaturatedError(RuntimeError)` |
| `agent/transports/codex_app_server.py` | `CodexAppServerError(RuntimeError)` |
| `cron/scheduler.py` | `CronSchedulerRegistrationError(RuntimeError)` |
| `gateway/platforms/api_server.py` | 内部 `_CronSchedulerRegistrationError`、`_ProviderAuthResolutionError` |
| `gateway/platforms/qqbot/chunked_upload.py` | `UploadDailyLimitExceededError`、`UploadFileTooLargeError` |
| `gateway/platforms/signal_rate_limit.py` | `SignalRateLimitError`、`SignalSchedulerError` |
| `gateway/turn_lease.py` | `TurnLeaseTimeoutError(TimeoutError)` |

## 3. 架构与约定

### 3.1 异常类型约定
- 自定义异常一律继承 Python 内置异常（`RuntimeError`、`ValueError`、`PermissionError`、`TimeoutError`、`Exception`），不定义新的根异常类。
- 命名遵循 `<Domain><Purpose>Error` 形式（如 `SignalRateLimitError`、`CompressionExecutorSaturatedError`、`CronSchedulerRegistrationError`）。
- 异常只携带必要上下文，不做复杂序列化——真正的结构化信息走 `error_classifier.ClassifiedError` dataclass。

### 3.2 API 错误分类与恢复
`classify_api_error(error, provider, model, approx_tokens, context_length, num_messages)` 是单点入口，按以下优先级运行：
1. Provider 特定模式（thinking signature、long-context tier gate、llama.cpp grammar、xAI subscription）
2. HTTP status code 分类（401/403→auth，402→billing，429→rate_limit/upstream_rate_limit，5xx→server_error/overloaded，413→payload_too_large，404→model_not_found）
3. error_code 分类（从 body 提取）
4. 消息体模式匹配（billing vs rate_limit vs auth vs context_overflow vs content_policy_blocked vs format_error）
5. SSL 证书验证失败 → 立即失败（`ssl_cert_verification`，`retryable=False`）
6. SSL 瞬态告警 → 当作 timeout 重试
7. 服务端断开 + 大会话 → context_overflow（触发压缩）
8. 传输/超时启发式 → timeout
9. 兜底 unknown（可重试）

分类结果通过 `retryable`、`should_compress`、`should_rotate_credential`、`should_fallback` 四个布尔字段驱动上层重试/回退/压缩逻辑。

### 3.3 网关致命错误
`BasePlatformAdapter` 维护 `_fatal_error_code`、`_fatal_error_message`、`_fatal_error_retryable` 三个属性，通过 `_set_fatal_error` 设置后调用 `_notify_fatal_error()` 通知注册的 handler。平台锁定冲突、认证解析失败等场景均通过此机制上报。

### 3.4 中间件错误传播
中间件执行链 `_run_execution_chain` 中：
- 下游异常被 `_DownstreamExecutionError` 包装后上抛，最终还原原始异常类型。
- 单个 middleware 回调抛错时记录 `warning`，若已调用过 `next_call()` 且成功则返回其结果，否则继续调用下一个回调。
- 重复调用 `next_call()` 抛出 `RuntimeError` 明确契约违规。

### 3.5 日志与错误可见性
- `errors.log` 固定 WARNING+，用于“快速排查”；`agent.log` 默认 INFO+，承载全部活动。
- 通过 `COMPONENT_PREFIXES` 将 `gateway.*`、`plugins.platforms` 路由到 `gateway.log`，`hermes_cli.web_server`、`tui_gateway.*`、`uvicorn.*` 路由到 `gui.log`。
- Windows 下使用 `concurrent_log_handler` 解决多进程同时写同一 log 文件的 `WinError 32` 问题；POSIX 下仍用 stdlib `RotatingFileHandler` 以兼容 NixOS managed mode 的 setgid 权限要求。
- 所有文件处理器经 `QueueListener` 在独立线程写入，避免阻塞事件循环；`handleError` 覆盖静默已知 Windows 并发锁超时。
- `RedactingFormatter` 在格式化阶段脱敏，确保 secrets 不落盘。

## 4. 约定与约束

- **禁止裸 `raise Exception`**：新异常应继承合适的内置类型，语义清晰（如 `PermissionError` 表示权限拒绝、`TimeoutError` 表示超时、`ValueError` 表示参数无效）。
- **上游 API 异常必须交给 `classify_api_error`**：不要在调用方自行字符串匹配判断是否重试，统一走分类器以保证行为一致。
- **平台适配器故障必须通过 `_set_fatal_error`**：不要直接 `logger.error` 后 return，需同步更新运行时状态并触发 fatal handler。
- **中间件不得多次调用 `next_call()`**：这是契约违规，会被显式抛出 `RuntimeError`。
- **日志不可泄露敏感信息**：所有文件处理器强制使用 `RedactingFormatter`，新增 logger 也必须通过 `setup_logging` 注册。
- **错误分类模式串必须窄而精确**：分类器注释明确要求每条模式串来自真实 provider 响应，避免误匹配导致健康会话进入压缩/回退死循环。
- **SSL 证书验证失败不重试**：分类器将其标记为 `retryable=False`，因为每次重试都会得到相同握手失败，应立即给出可操作指引。
- **日志写入失败不应中断主流程**：`_ManagedRotatingFileHandler.handleError` 对已知 Windows 锁超时报错静默吞掉；status 写入失败首次 warning 后降级 debug，避免重连风暴刷屏。

## 5. 适用性说明

本仓库是一个大型多入口聚合系统（Python agent + Gateway + CLI + TUI + Web + 工具/技能/插件/Cron/ACP），错误处理贯穿所有子系统，但**没有单一全局异常基类或统一中间件框架**。错误处理以“领域异常 + 中心化分类器 + 统一日志基础设施”的组合方式实现，属于中等成熟度（medium-high）的体系化实践。