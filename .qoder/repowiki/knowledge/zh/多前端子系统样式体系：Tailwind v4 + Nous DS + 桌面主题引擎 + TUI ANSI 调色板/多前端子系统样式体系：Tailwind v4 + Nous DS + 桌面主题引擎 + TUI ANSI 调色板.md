---
kind: frontend_style
name: 多前端子系统样式体系：Tailwind v4 + Nous DS + 桌面主题引擎 + TUI ANSI 调色板
category: frontend_style
scope:
    - '**'
source_files:
    - web/src/index.css
    - web/vite.config.ts
    - web/src/themes/index.ts
    - web/src/themes/context.tsx
    - web/src/themes/presets.ts
    - apps/desktop/src/styles.css
    - apps/desktop/src/themes/presets.ts
    - apps/desktop/src/themes/skin.ts
    - apps/desktop/src/themes/context.tsx
    - ui-tui/src/theme.ts
---

## 1. 总体方案

仓库包含三个独立的前端子系统，各自采用不同的样式/主题策略，但共享同一套设计语言（Nous Design System）与品牌语义。

| 子系统 | 框架 | 样式系统 | 主题机制 |
|---|---|---|---|
| Web Dashboard (`web/`) | React + Vite | Tailwind CSS v4 (`@tailwindcss/vite` 插件) + `@nous-research/ui` 设计系统 | CSS 变量 + `ThemeProvider` 动态注入；默认 LENS_0 (Hermes Teal) |
| Desktop App (`apps/desktop/`) | Electron + React + Vite | Tailwind CSS v4 + `@tailwindcss/typography` + `tw-shimmer` + `katex` + `codicon` | 自研主题引擎 (`src/themes/`)：种子色 → `color-mix` 派生层 → shadcn/DaT 语义 token 映射 |
| TUI (`ui-tui/`) | Ink (Node.js 终端 UI) | 无 CSS；ANSI 256 / truecolor 输出 | `theme.ts`：种子色 → 派生色调阶梯 → 对比度自适应 → ANSI 归一化 |

## 2. 关键文件与包

- **Web 入口**：`web/src/index.css` — 通过 `@import 'tailwindcss'`、`@source '../node_modules/@nous-research/ui/dist'` 扫描 DS 组件类名，定义 LENS_0 的 `--foreground/--midground/--background` 等根变量，并桥接 shadcn 兼容 token（`--color-card`, `--color-muted-foreground` 等）。
- **Web 主题上下文**：`web/src/themes/context.tsx`、`presets.ts`、`fonts.ts`、`types.ts` — 提供 `ThemeProvider`/`useTheme`、内置主题列表、字体选择器。
- **Vite 构建**：`web/vite.config.ts` — 使用 `@tailwindcss/vite` 插件，按 `react-vendor/xterm/three/plot/motion/ui/vendor` 分组 code-splitting，构建产物输出到 `hermes_cli/web_dist`。
- **Desktop 全局样式**：`apps/desktop/src/styles.css` — 定义 `:root` / `:root.dark` 两套完整 token 树（`--dt-*` DaT 语义层、`--ui-*` 应用层、`--z-*` z-index 阶梯），并通过 `@custom-variant dark (&:is(.dark *))` 和 `compact` 媒体查询扩展变体。同时声明 `prefers-reduced-motion` 全局禁用动画。
- **Desktop 主题引擎**：`apps/desktop/src/themes/` 目录
  - `presets.ts`：内置主题（`nous`/`midnight`/`ember`/`mono`/`cyberpunk`/`slate`），每个主题含 light/dark 两套颜色与可选 Google Fonts URL。
  - `skin.ts`、`user-themes.ts`、`backend-sync.ts`：用户皮肤、后端同步、VSCode 集成。
  - `context.tsx`：React 主题上下文，运行时切换皮肤。
  - `install.ts`：将主题字体下载到本地 `fonts/` 目录。
- **TUI 主题**：`ui-tui/src/theme.ts` — 定义 `ThemeSeeds` → `buildPalette` → `deriveTones` 的派生阶梯，实现 WCAG 对比度保底（显示色 ≥1.45/2.2，语义色 ≥2.2）、填充极性保护、Apple Terminal 的 ANSI 256 归一化（`normalizeThemeForAnsiLightTerminal`）。

## 3. 架构与设计约定

### 3.1 设计令牌分层（Desktop 模式，被 Web 借鉴）

Desktop 的 `styles.css` 建立了三层 token 体系：
1. **种子层** (`--theme-*`)：`--theme-primary`、`--theme-background-seed`、`--theme-mix-*` 混合比例等原始身份色。
2. **应用层** (`--ui-*`)：由种子经 `color-mix(in srgb, ...)` 派生的具体表面色，如 `--ui-bg-chrome`、`--ui-text-primary`、`--ui-chat-bubble-background`。
3. **语义层** (`--dt-*`)：面向组件的 shadcn/DaT 兼容命名，如 `--dt-background`、`--dt-primary`、`--dt-input-border`，供 Tailwind 工具类直接消费。

z-index 同样分层为 `--z-modal-backdrop(120)` → `--z-over-modal(200)` → `--z-connecting(1200)` 等固定台阶，禁止随意写字面量。

### 3.2 Web Dashboard 的主题策略

- 默认主题为 LENS_0（Hermes Teal），在 `index.css` 中用 `:root` 变量声明，由 `ThemeProvider` 以 inline style 重写。
- 通过 `@source '../node_modules/@nous-research/ui/dist'` 让 Tailwind JIT 扫描 DS 组件源码中的类名，避免被 purge 掉。
- 旧 shadcn 调用点通过 `@theme inline` 重映射到 DS 语义 token，无需逐处改写。

### 3.3 TUI 的“种子→派生”模型

TUI 的 `theme.ts` 把 desktop 的 `color-mix` 理念移植到 ANSI 环境：
- 种子色（`DARK_SEEDS`/`LIGHT_SEEDS`）只定义 identity 色（primary/accent/text/border/status）。
- 所有次要色（muted/label/surface/selection）由 `deriveTones()` 根据背景亮度自动推导。
- 通过 `adaptColorsToBackground()` 强制对比度底线，错误极性的填充回退到派生值。
- 对 Apple Terminal 的 256 色限制，通过 `bestReadableAnsiColor()` 映射到最接近的 ansi256 编号。

### 3.4 字体策略

- Web：`@nous-research/ui` 的 `fonts.css` 注册 Collapse/Mondwest 字体，JetBrains Mono 通过 `@font-face` 从 `/fonts-terminal/` 静态资源加载。
- Desktop：`Collapse` 字体从 `@nous-research/ui/dist/fonts/` 引入，JetBrains Mono 从本地 `./fonts/` 加载；`presets.ts` 中各主题可指定 Google Fonts URL（Courier Prime、JetBrains Mono、IBM Plex Mono）。
- TUI：依赖宿主终端字体，通过 OSC 10/11 读取终端背景色做对比度适配。

## 4. 约定与约束

- **Tailwind v4 统一**：Web 与 Desktop 均使用 `tailwindcss@4.3.3` + `@tailwindcss/vite`，不再需要 `tailwind.config.*` 文件；配置通过 `@theme inline` 块内联声明。
- **设计系统优先**：Web 通过 `@nous-research/ui` 获取基础组件与字体；Desktop 通过 `@assistant-ui/react`、`radix-ui`、`cmdk` 等组合出高级组件，但颜色全部走 `--dt-*` 语义层，确保换肤时一致。
- **响应式与无障碍**：Desktop 在 `styles.css` 顶部通过 `@media (prefers-reduced-motion: reduce)` 全局禁用动画；Web 在 `index.css` 中对 `<768px` 设备调整 `dvh` 布局。
- **RTL 支持**：Web 通过 `html[dir="rtl"]` 作用域规则启用逻辑间距（`ms-/me-`、`ps-/pe-`），配合 i18n provider 设置 `dir`。
- **主题不可破坏性**：Desktop 的 z-index 阶梯、输入边框强度（`--dt-input-border` 仅 7% 不透明度）、阴影层级（`--shadow-nous` 多层叠加）均为受控常量，新增主题不得绕过这些约束。
- **TUI 对比度底线**：`theme.ts` 中显式定义 `DISPLAY_MIN_CONTRAST=1.45`、`SEMANTIC_MIN_CONTRAST=2.2`，任何 skin 都不能输出低于此阈值的文本色。

## 5. 相关路径

- `web/src/index.css` — Web 全局样式与 LENS_0 主题变量
- `web/vite.config.ts` — Tailwind v4 + 代码分割策略
- `web/src/themes/` — Web 主题上下文与预设
- `apps/desktop/src/styles.css` — Desktop 全局样式与 token 树
- `apps/desktop/src/themes/presets.ts` — 内置主题定义
- `apps/desktop/src/themes/skin.ts` — 用户皮肤管理
- `ui-tui/src/theme.ts` — TUI 主题引擎与 ANSI 适配