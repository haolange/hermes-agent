---
kind: configuration_system
name: Hermes Agent 配置系统：YAML + .env + 多源合并与迁移
category: configuration_system
scope:
    - '**'
source_files:
    - hermes_cli/config.py
    - hermes_cli/config_defaults.py
    - hermes_cli/fallback_config.py
    - gateway/config.py
    - cli-config.yaml.example
    - hermes_cli/config_migrations.py
    - hermes_constants.py
    - docker/entrypoint.sh
    - docker/main-wrapper.sh
---

## 1. 采用的系统与方案

Hermes Agent 使用**自研的 Python 配置加载器**，以 `~/.hermes/config.yaml` 为主配置文件、`~/.hermes/.env` 为密钥文件，通过 `hermes_cli/config.py` 实现 YAML/环境变量/默认值/迁移脚本的多层合并。Gateway 侧在 `gateway/config.py` 中再对平台配置做 dataclass 化校验与运行时覆盖（如 `GATEWAY_MULTIPLEX_PROFILES`）。

- **配置文件格式**：YAML（主配置）+ `.env`（键值对，支持 `${VAR}` 引用）。
- **默认值来源**：`hermes_cli/config_defaults.py` 中的 `DEFAULT_CONFIG` 字典（纯数据模块，不反向导入 config），以及 `cli-config.yaml.example` 作为用户可复制的完整示例。
- **环境变量优先级**：`.env` 中的变量优先于 `config.yaml` 对应项；另有进程级环境变量（如 `HERMES_HOME`、`HERMES_PROFILE`、`HERMES_ENV`、`HERMES_MANAGED`、`GATEWAY_*`）可覆盖行为。
- **多源合并**：`load_config()` 将 `DEFAULT_CONFIG`、用户 `config.yaml`、`managed-scope` 配置（NixOS/Homebrew）、`.env` 展开后的值进行深合并，并缓存结果（基于路径+mtime+size 的 `_LOAD_CONFIG_CACHE`）。
- **迁移系统**：`hermes_cli/config_migrations.py` 提供按版本号的配置迁移钩子，`ENV_VARS_BY_VERSION` 记录每个版本新增的环境变量以便提示用户。
- **受管模式（Managed Mode）**：通过 `HERMES_MANAGED` 或 `$HERMES_HOME/.managed` 标记 NixOS/Homebrew 等包管理器安装，禁止直接写 `config.yaml`，改为由激活脚本管理目录权限（0750/0640）。

## 2. 关键文件与包

- `hermes_cli/config.py`：核心配置加载/保存/迁移/安全策略（目录权限、容器检测、denylist 环境写入保护）。
- `hermes_cli/config_defaults.py`：`DEFAULT_CONFIG` 与 `OPTIONAL_ENV_VARS` 纯数据定义。
- `hermes_cli/fallback_config.py`：解析 `fallback_providers` / `fallback_model` 链，统一旧/新 key 的 fallback 入口。
- `gateway/config.py`：Gateway 平台配置的 dataclass 模型（`PlatformConfig`、`StreamingConfig`、`SessionResetPolicy`、`GatewayConfig`），含类型强制、连接检查器、端口绑定判定。
- `cli-config.yaml.example`：完整的用户配置示例文档（含 model、terminal、compression、streaming、skills、agent 等所有章节注释）。
- `hermes_constants.py`：`get_hermes_home()` 等路径常量（被 config.py 重新导出）。
- `hermes_cli/config_migrations.py`：配置版本迁移逻辑。
- `docker/entrypoint.sh`、`docker/main-wrapper.sh`：容器内设置 `HERMES_UID/GID`、`HERMES_SKIP_CHMOD` 等运行期环境变量。
- `nix/*.nix`：NixOS 模块声明式配置（通过 `services.hermes-agent.settings` 注入）。

## 3. 架构与约定

### 配置加载顺序（从高到低）
1. 进程环境变量（`os.environ`，或通过 `agent.secret_scope.get_secret` 读取 profile-scoped secret）。
2. `~/.hermes/.env` 文件（支持 `${VAR}` 展开）。
3. `~/.hermes/config.yaml`（用户覆盖）。
4. Managed-scope 配置（NixOS 激活脚本写入）。
5. `DEFAULT_CONFIG`（代码内置默认值）。

### 安全约定
- **目录/文件权限**：非 managed 模式下 `ensure_hermes_home()` 创建 `~/.hermes` 及子目录时设为 `0o700`，`.env` 等敏感文件设为 `0o600`；容器内跳过 chmod（`_is_container()` 检测 `/proc/1/cgroup`、`/.dockerenv`、`HERMES_CONTAINER` 等）。
- **`.env` 写入 denylist**：`save_env_value()` 拒绝写入 `LD_PRELOAD`、`PYTHONPATH`、`PATH`、`EDITOR`、`HERMES_HOME`、`HERMES_PROFILE` 等危险变量名（白名单式允许 `HERMES_*` 集成密钥）。
- **损坏配置恢复**：YAML 解析失败时自动备份到 `config.yaml.corrupt.<ts>.bak`，并回退到 `DEFAULT_CONFIG`，同时仅告警一次（基于 mtime+size 去重）。
- **原子写入**：`utils.atomic_replace` 保证 `save_config()` 不会留下半写文件。

### 多实例与 Profile
- 通过 `HERMES_HOME` 切换不同 profile 的配置空间（`profiles/<name>/`）。
- Gateway 支持 multiplex profiles（`GATEWAY_MULTIPLEX_PROFILES=1`），单进程服务多个 profile，通过 URL prefix `/p/<profile>/` 路由。

### 安装方式检测
`detect_install_method()` 按以下顺序识别：代码级 `.install_method` 戳 → 历史 home 级戳 → `HERMES_MANAGED`/`.managed` → `/nix/store` 路径 → `.git` → `unknown`，用于决定更新命令和是否允许写配置。

## 4. 约定与约束

- **配置位置固定**：主配置必须在 `~/.hermes/config.yaml`，密钥在 `~/.hermes/.env`，不可通过命令行参数随意指定（`HERMES_CONFIG` 仅影响 CLI 内部行为，不被 dashboard 写入）。
- **受管安装禁止直接修改**：当 `is_managed()` 为真时，任何写 `config.yaml` 的操作都会报错并提示通过 NixOS/Homebrew 管理。
- **YAML 解析失败即降级**：解析异常不会抛错中断启动，而是静默回退到默认配置并告警，便于生产环境容灾。
- **环境变量命名规范**：所有集成密钥遵循 `HERMES_*` 前缀（如 `HERMES_LANGFUSE_*`、`HERMES_ACP_*`），但 loader 对少数危险变量名显式 denylist。
- **Gateway 平台枚举严格**：`Platform` 枚举只接受内置平台或已注册的插件平台名称，未知字符串会被拒绝（防止任意字符串污染配置）。
- **配置变更热重载**：MCP 服务器发现等部分配置支持 `auto_reload_on_config_change`，但会失效 provider prompt cache（设计权衡）。
- **向后兼容**：`fallback_providers` 与旧的 `fallback_model` 共存，`get_fallback_chain()` 自动合并去重；`ENV_VARS_BY_VERSION` 记录新增 env var 以便迁移提示。
- **容器部署约定**：通过 `HERMES_UID`/`HERMES_GID` 控制 chown，`HERMES_SKIP_CHMOD=1` 跳过权限调整，`HERMES_CONTAINER` 显式标记容器环境。