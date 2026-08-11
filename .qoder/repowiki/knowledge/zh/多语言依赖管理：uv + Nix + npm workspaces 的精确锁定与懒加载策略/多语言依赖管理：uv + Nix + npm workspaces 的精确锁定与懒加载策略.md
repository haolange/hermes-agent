---
kind: dependency_management
name: 多语言依赖管理：uv + Nix + npm workspaces 的精确锁定与懒加载策略
category: dependency_management
scope:
    - '**'
source_files:
    - pyproject.toml
    - uv.lock
    - package.json
    - .npmrc
    - website/.npmrc
    - flake.nix
    - flake.lock
    - nix/packages.nix
    - .github/dependabot.yml
---

## 1. 使用的系统与方法

Hermes Agent 是一个多语言仓库，Python、Node.js、Nix 三套依赖管理系统并存且互相协作：

- **Python**：使用 `pyproject.toml` + `uv.lock`（uv 包管理器）进行依赖声明与解析锁定。构建后端为 setuptools，但实际安装/开发由 uv 驱动。
- **Node.js**：使用 npm workspaces（根 `package.json` 中声明 `apps/*`、`ui-tui`、`web`、`tests-js` 等 workspace），每个子项目独立 `package.json`，根目录提供统一脚本与审计命令。通过 `overrides` 强制覆盖已知漏洞依赖版本。
- **Nix**：通过 `flake.nix` + `flake.lock` 声明上游 flake 输入（nixpkgs、pyproject-nix、uv2nix、npm-lockfile-fix 等），并使用 `uv2nix` 将 Python 依赖映射到 Nix store，实现可重现的 sealed venv。
- **CI/自动化**：GitHub Dependabot 仅对 `github-actions` 启用自动更新；Python/JS 源码依赖不启用自动 bump，完全走人工评审流程。

## 2. 关键文件

- `pyproject.toml`：Python 核心依赖、可选 extras、build-system、tool 配置集中声明处。
- `uv.lock`：uv 生成的完整依赖解析快照，所有 Python 依赖在此精确锁定。
- `package.json`（根）：npm workspaces 入口，定义 workspace、scripts、overrides、engines。
- `.npmrc`（根）：全局 npm 配置，启用 `engine-strict=true` 与 `min-release-age=14`，并通过 `min-release-age-exclude[]` 临时放行若干安全修复包。
- `website/.npmrc`：网站子项目的 npm 配置副本。
- `flake.nix` / `flake.lock`：Nix flake 输入声明与锁定。
- `nix/packages.nix`：通过 `extraDependencyGroups` 组合不同 Nix package profile（minimal/full/messaging/tui/web/desktop）。
- `.github/dependabot.yml`：Dependabot 策略文档化——仅 github-actions 生态启用自动 PR。

## 3. 架构与约定

### Python 依赖分层

`pyproject.toml` 将依赖分为三层：

1. **`dependencies`（核心）**：每个 session 都需要的包，全部使用 `==X.Y.Z` 精确锁定，不允许范围约束。注释明确说明这是供应链安全策略——范围约束会让 PyPI 在无人审查时引入新版本的传递依赖。
2. **`[project.optional-dependencies]`（extras）**：按功能域切分（`anthropic`、`messaging`、`matrix`、`voice`、`wake`、`google`、`web` 等），按需安装。这些包通过 `tools/lazy_deps.py` 在首次使用时懒加载安装，避免污染默认安装面。
3. **`[all]` extra**：刻意精简，只包含无法懒安装的依赖（CLI、pty、mcp、homeassistant、sms、acp、google、web、youtube）。注释明确排除了 provider/search/TTS/memory/messaging 等 opt-in 后端，防止上游被隔离的发布破坏全新安装。

平台标记广泛使用 `sys_platform == 'win32'`、`platform_system == 'Darwin'`、`('android' not in platform_release)` 等条件表达式，使同一份 `pyproject.toml` 在不同平台上解析出不同的依赖集。

### Node.js 依赖管理

- 根 `package.json` 通过 `workspaces` 聚合多个前端子工程。
- 通过 `overrides` 强制覆盖已知漏洞依赖（lodash、yauzl、protobufjs、brace-expansion）。
- `.npmrc` 设置 `engine-strict=true` 强制 Node/npm 版本匹配，并启用 `min-release-age=14` 要求依赖发布至少 14 天才能安装，配合大量 `min-release-age-exclude[]` 条目为安全修复包开绿灯。
- `allowScripts` 白名单控制允许运行的 postinstall 脚本。

### Nix 集成

`flake.nix` 通过 pyproject-nix + uv2nix 将 Python 依赖纳入 Nix 世界，`nix/packages.nix` 用 `extraDependencyGroups` 叠加不同 extras 构建出 minimal/full/messaging 等不同 Nix package profile。这使 Nix 用户可以直接 `nix profile install .#full` 获得预装所有可选功能的 hermes-agent。

### 懒加载机制

`tools/lazy_deps.py` 是运行时懒安装的核心：当用户选择某个 provider（如 anthropic/exa/firecrawl/fal/edge-tts/honcho/hindsight/supermemory/mem0/mistral/otlp 等）时，代码会在首次使用时触发 `pip install` 或 `uv pip install` 安装对应 extra。`pyproject.toml` 中的 `[all]` 故意不包含这些包，确保即使上游某包被隔离也不会阻断新用户安装。

## 4. 约定与约束

- **Python 禁止范围约束**：`pyproject.toml` 注释明确要求所有核心依赖必须使用 `==X.Y.Z` 精确锁定，不得引入 `>=x,<y` 范围。更新时必须同步修改版本号并重新运行 `uv lock` 生成一致的 `uv.lock`。
- **供应链隔离策略**：`[all]` 仅包含真正无法懒安装的依赖；opt-in 后端（provider、search、TTS、image、memory、messaging platform、terminal sandbox）必须走 `tools/lazy_deps.py`，这样上游被隔离的发布不会破坏 fresh install。
- **最小暴露面**：只有被每个 hermes session 直接使用的包才放入核心 `dependencies`；provider-specific 包放入 extra 并懒加载，减少供应链攻击面。
- **Node.js 版本锁定**：`.npmrc` 的 `engine-strict=true` 强制 Node >=22.22.0 且 npm <11.10.0 || >=11.17.0；`min-release-age=14` 阻止刚发布的依赖进入构建。
- **Dependabot 仅限 GitHub Actions**：`.github/dependabot.yml` 明确声明不对 pip/npm 启用自动 bump，因为源码依赖以精确锁定的方式管理，版本移动需经人工评审。
- **安全更新例外**：Dependabot security updates（仅在 CVE 公开时触发）仍可用，这与 pinning 策略不冲突。
- **Nix 可重现性**：`flake.lock` 锁定所有 flake 输入版本；`nix/packages.nix` 通过 `rev = inputs.self.rev or null` 嵌入干净 commit rev，避免 dirtyRev 导致“有更新”误报。
- **构建系统锁定**：`pyproject.toml` 的 `[build-system].requires` 锁定 setuptools==83.0.0，避免构建期工具版本漂移影响打包产物。
- **uv 额外约束**：`exclude-newer = "14 days"` 限制 uv 拉取新包的时间窗口；`exclude-newer-package` 为 vercel/nemo-relay/huggingface_hub 等特定包豁免该限制。