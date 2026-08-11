---
kind: build_system
name: Hermes Agent 多语言构建与发布系统（uv + Docker + Nix + GitHub Actions）
category: build_system
scope:
    - '**'
source_files:
    - pyproject.toml
    - uv.lock
    - Dockerfile
    - .github/workflows/ci.yml
    - .github/workflows/tests.yml
    - .github/workflows/docker.yml
    - flake.nix
    - nix/packages.nix
    - scripts/release.py
    - scripts/run_tests.sh
    - scripts/install.sh
    - docker/entrypoint-dispatch.sh
    - docker/stage2-hook.sh
---

## 1. 构建系统与工具链

Hermes Agent 是一个 Python+Node.js 混合仓库，采用 **分层、可复现** 的构建体系：

- **Python 依赖管理**：使用 `pyproject.toml` + `uv.lock` 锁定全部依赖版本（所有核心依赖精确到 `==X.Y.Z`，见 pyproject 注释），通过 `uv sync --locked` 在 CI 和开发环境中安装。可选功能按 `[project.optional-dependencies]` 拆分为 extra（如 `anthropic`、`messaging`、`matrix`、`voice`、`web`、`all` 等），生产镜像仅安装精选的 `--extra all --extra messaging --extra otlp --extra anthropic ...` 子集，避免供应链攻击面。
- **前端构建**：根 `package.json` + `package-lock.json` 管理 monorepo workspace（root/web/ui-tui/apps/shared），通过 `npm install` 安装；`web/` 和 `ui-tui/` 各自用 Vite 构建产物（`npm run build`）。Node 版本固定为 26（Dockerfile 中从 `node:26-bookworm-slim` 复制二进制，注释说明这是为了 glibc 兼容性）。
- **Nix 打包**：`flake.nix` 引入 `pyproject-nix` + `uv2nix`，将 Python 依赖编译进 Nix store；`nix/packages.nix` 定义 `minimal`、`full`（含所有 optional extras）、`messaging`、`tui`、`web`、`desktop`、`sandbox` 等多个 package，支持跨 `x86_64-linux` / `aarch64-linux` / `aarch64-darwin` 三架构。
- **Docker 镜像**：`Dockerfile` 是核心产物，基于 Debian 13 (trixie)，包含预编译 SQLite 3.53.4（修复 WAL-reset bug）、s6-overlay 进程监督、Node 26、Playwright Chromium、以及 hermes-agent 的可编辑安装。镜像以非 root 用户 `hermes` 运行，数据卷挂载于 `/opt/data`。
- **安装器**：`scripts/install.sh` 提供 Linux/macOS/Termux 的一键安装，默认使用 uv 创建 venv，支持 FHS 布局（root 下 `/usr/local/lib/hermes-agent` + `/usr/local/bin/hermes`）。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `pyproject.toml` | Python 包元数据、依赖 pin、extras、entry points (`hermes`, `hermes-agent`, `hermes-acp`) |
| `uv.lock` | 锁定的完整依赖图 |
| `Dockerfile` | 多阶段构建：sqlite_build → uv_source → node_source → 最终镜像 |
| `.github/workflows/ci.yml` | 总编排：detect-changes → tests/lint/js/docker/supply-chain → all-checks-pass |
| `.github/workflows/tests.yml` | 测试分片并行执行（LPT 切分 + 缓存时长） |
| `.github/workflows/docker.yml` | 多架构 Build/Publish/Merge（amd64/arm64），main→`:main/:latest`，release→`:<tag>` |
| `flake.nix` + `nix/*.nix` | Nix flake 定义，uv2nix 生成封闭 venv |
| `scripts/release.py` | CalVer 风格发布脚本（`--bump minor --publish`），生成 changelog 并打 tag |
| `scripts/run_tests.sh` | 统一测试入口，强制隔离环境（TZ=UTC, LANG=C.UTF-8, PYTHONHASHSEED=0） |
| `scripts/install.sh` | 跨平台安装器 |
| `docker/entrypoint-dispatch.sh` + `docker/stage2-hook.sh` | 容器运行时调度（PID 1 路径与非 PID 1 回退） |

## 3. 架构与约定

### 依赖策略
- **核心依赖必须精确 pin**（`==X.Y.Z`），范围约束仅在极少数场景（如 `urllib3>=2.7.0,<3`）允许，且需注释理由。
- **可选后端走 lazy-install**：`tools/lazy_deps.py` 在首次使用时按需安装（如 firecrawl-py、exa-py、elevenlabs），不进入 `[all]`，避免上游被污染时破坏全量安装。
- **Python 版本**：`requires-python = ">=3.11,<3.14"`，CI 使用 3.11，Nix devShell 使用 3.13。

### 构建分层
Dockerfile 严格分层以最大化缓存命中：
1. 系统依赖层（apt-get）
2. SQLite 自定义编译层
3. s6-overlay 安装层
4. Node/npm 安装层
5. Python 依赖层（只拷贝 `pyproject.toml` + `uv.lock`）
6. 前端构建层（只拷贝 web/ui-tui 源码）
7. 源代码层（`COPY . .`，带 `--chmod=a+rX,go-w` 直接标记只读）
8. 可编辑安装层（`uv pip install -e .`）
9. 运行时配置层（写入 `.install_method`、`.hermes_build_sha`）

### 多架构支持
- Docker：BuildKit 自动识别 `TARGETARCH`，分别构建 linux/amd64 与 linux/arm64，最后 `imagetools create` 合并为 manifest list。
- Nix：`flake.nix` 声明 `systems = [ "x86_64-linux" "aarch64-linux" "aarch64-darwin" ]`。
- s6-overlay 通过 `case ${TARGETARCH}` 选择对应 tarball。

### 版本与发布
- 版本号来自 `pyproject.toml` 的 `version = "0.20.0"`。
- 发布流程：`python scripts/release.py --bump minor --publish` 生成 CalVer 标签（如 `2026.3.15`），触发 release workflow 推送 Docker Hub。
- Git SHA 通过 `HERMES_GIT_SHA` build-arg 注入镜像，运行时由 `hermes_cli/build_info.py` 读取 `/opt/hermes/.hermes_build_sha`。

### 测试策略
- 单入口：`scripts/run_tests.sh` 强制隔离每个测试文件（独立 python 子进程），禁止模块级状态泄漏。
- CI 分片：`tests.yml` 先 `--generate-slices` 计算 LPT 分布，再并行跑多个 slice。
- 环境变量白名单：`env -i` 启动，仅放行 PATH/HOME/WIN_ENV/TEST_ENV 中的显式变量。
- 集成测试：`tests/docker/` 针对已构建镜像运行，通过 `HERMES_TEST_IMAGE` 复用本地镜像避免重复构建。

## 4. 约定与约束

- **依赖变更必须同步更新 `uv.lock`**：`uv sync --locked` 会在 CI 失败，确保 lock 与 `pyproject.toml` 一致。
- **新增可选后端必须走 lazy-install**：不得加入 `[all]`，否则一个被污染的 PyPI 包会阻断所有新用户安装。
- **Docker 镜像不可写**：`/opt/hermes` 在 COPY 后标记 `go-w`，运行时 lazy-install 目标重定向到 `/opt/data/lazy-packages`。
- **CI 分支保护**：`ci.yml` 的 `all-checks-pass` 是唯一要求通过的检查，其他 job 通过 `if: needs.detect.outputs.* == 'true'` 条件跳过。
- **安全边界**：PR 触发的 workflow 不能访问 secrets（只有 main push/release 的 `container-publish` environment 才能登出 Docker Hub）。
- **Node 版本锁定**：Dockerfile 注释明确“Bumping to a new Node major is a one-line ARG change”，当前锁定 26 以避免 EOL。
- **SQLite 版本强制升级**：构建期编译 3.53.4 并替换系统 lib，运行时校验 `sqlite_version_info >= (3, 51, 3)`，否则 abort。
- **贡献者映射集中化**：`scripts/release.py` 中的 `LEGACY_AUTHOR_MAP` 不再新增新条目，新映射放在 `contributors/emails/` 下一文件一邮箱目录。
- **测试必须无外部 API 调用**：CI 中 `OPENROUTER_API_KEY`、`OPENAI_API_KEY`、`NOUS_API_KEY` 均设为空字符串，`tests/conftest.py` 设置 `HERMES_DISABLE_LAZY_INSTALLS=1` 禁止运行时安装。