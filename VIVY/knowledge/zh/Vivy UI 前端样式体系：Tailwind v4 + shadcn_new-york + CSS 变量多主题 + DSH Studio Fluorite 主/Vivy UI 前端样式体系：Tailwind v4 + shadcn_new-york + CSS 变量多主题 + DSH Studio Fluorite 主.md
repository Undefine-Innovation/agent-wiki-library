---
kind: frontend_style
name: Vivy UI 前端样式体系：Tailwind v4 + shadcn/new-york + CSS 变量多主题 + DSH Studio Fluorite 主题
category: frontend_style
scope:
    - '**'
source_files:
    - ui/src/styles.css
    - ui/components.json
    - ui/package.json
    - ui/vite.config.ts
    - studio/dsh-vivy-studio/theme.css
    - studio/dsh-vivy-studio/package.json
    - studio/dsh-vivy-studio/check-tokens.mjs
---

## 1. 系统概览

本仓库包含两套独立的前端样式体系，分别服务于不同的运行时产物：

- **Vivy UI（`ui/`）**：基于 React + Vite + Tailwind CSS v4 + shadcn/ui (new-york 风格) 构建的 Web 界面，通过 Go 嵌入 (`embed.go`) 随 `vivy.exe` 分发。
- **Vivy Studio（`studio/dsh-vivy-studio/`）**：作为 DSH (Dive Studio Host) web shell 的主题包，提供品牌色、CSS 变量覆盖与文案替换，以独立 npm 包形式注入宿主。

两者均使用 CSS 自定义属性（CSS Variables）作为设计令牌载体，实现主题切换与品牌定制。

## 2. 关键文件与依赖

| 位置 | 作用 |
|---|---|
| `ui/package.json` | 声明 Tailwind v4 (`@tailwindcss/vite`, `tailwindcss@^4.2.1`)、Radix UI 全量组件、shadcn 工具链 (`class-variance-authority`, `clsx`, `tailwind-merge`)、Lucide 图标、Framer Motion 动画等 |
| `ui/src/styles.css` | 全局样式入口：Tailwind 导入、`@theme inline` 令牌映射、四套主题 (`:root` / `[data-theme="dark"]` / `[data-theme="pink"]` / `[data-theme="miku"]`)、`base` layer 重置 |
| `ui/components.json` | shadcn/ui 配置：style=`new-york`、baseColor=`slate`、CSS 变量开启、iconLibrary=`lucide`、路径别名 (`@/components/ui` → shadcn 基础组件库) |
| `ui/vite.config.ts` | 构建配置：启用 `@tailwindcss/vite`、固定 dev 端口 3015、HMR 默认关闭（沙箱 iframe 场景）、输出到 `dist/assets/[name]-[hash].*` |
| `studio/dsh-vivy-studio/theme.css` | DSH 主题：定义 `--vivy-fluorite` 系列品牌色与 `--dsw-alias-*` / `--dsw-specific-*` 全部 DSH 令牌，含 light/dark 双态；通过 CSS `::before` / `::after` 伪元素替换 DSH 内置 logo 与欢迎文案 |
| `studio/dsh-vivy-studio/package.json` | 声明为 DSH bundle，附带 `cordis.patch.yml` 补丁清单 |
| `studio/dsh-vivy-studio/check-tokens.mjs` | 校验 DSH 令牌是否被完整覆盖的脚本 |

## 3. 架构与设计约定

### 3.1 设计令牌分层

- **UI 层**：`src/styles.css` 中 `@theme inline { ... }` 将 shadcn 语义化 token（`--color-primary`、`--color-card`、`--radius-md` 等）映射到底层 CSS 变量，使 Tailwind 类名与变量解耦。
- **主题层**：每个主题以 `[data-theme="..."]` 选择器集中声明一组完整的 `--background` / `--foreground` / `--primary` / `--chart-*` / `--sidebar-*` 等变量。当前内置四套：
  - `:root`（浅蓝灰，default）
  - `[data-theme="dark"]`（深蓝夜色，注释称 default 的深色版）
  - `[data-theme="pink"]`（简约粉白，移植自 Agent-Diva default 主题）
  - `[data-theme="miku"]`（Miku 青，GitHub Dark 底 + 应援青）
- **颜色空间**：默认主题使用 OKLCH 色彩空间 (`oklch(...)`)，保证感知均匀性；部分主题（如 pink/miku）保留十六进制以保持与原 Agent-Diva 一致。

### 3.2 暗色模式策略

- 使用 `@custom-variant dark (&:is(.dark *));` 定义 Tailwind 的 `dark:` 变体。
- 注释明确：`.dark` 类仅驱动 `dark:` 变体，不承载令牌值；实际令牌由 `data-theme` 属性控制，预览色板字面量在 `src/hooks/use-theme.ts` 中管理。

### 3.3 字体策略

- UI 默认字体栈：`PingFang SC` → `Microsoft YaHei` → `Hiragino Sans GB` → system sans-serif，优先中文无衬线。
- Studio 主题字体：`Source Han Serif SC` / `Noto Serif SC` / `Songti SC` 用于标题显示，`Segoe UI` / `PingFang SC` / `Hiragino Sans GB` 用于正文。

### 3.4 Studio 主题注入机制

`theme.css` 通过覆盖 DSH 框架暴露的 `--dsw-alias-*` 与 `--dsw-specific-*` 变量完成品牌化，并采用纯 CSS 手段替换 DSH 原生内容：
- 隐藏 DSH whale logo SVG（`svg[viewBox="0 0 182 24"] { width:0; height:0; position:absolute }`），用 `::before` 伪元素绘制 "V" 徽标 + "Nierland" 文字。
- 重写欢迎页标题为 "Vivy Studio"，重写说明文案为 "Vivy Studio 是独立应用..."。
- 隐藏 fishHitbox 与特定蓝色椭圆，统一为 fluorite 青色。

### 3.5 动画与交互

- 引入 `tw-animate-css` 提供 Tailwind 兼容的预置动画。
- 依赖 `framer-motion` 提供复杂交互动画。
- 尊重 `prefers-reduced-motion`：滚动行为强制为 `auto`。

## 4. 约束与规范

- **Tailwind v4 语法**：使用 `@import "tailwindcss" source(none)` 与 `@source "../src"` 进行源码扫描，而非传统 `tailwind.config.js` 配置文件。
- **shadcn 组件来源**：所有基础 UI 组件来自 `@/components/ui`（即 shadcn 生成的 Radix 封装），禁止直接手写基础控件样式。
- **变量命名**：所有可主题化的视觉值必须走 CSS 变量，禁止在组件 JSX 中硬编码颜色/圆角/字号。
- **Studio 主题完整性**：`check-tokens.mjs` 会校验 DSH 主题变量是否被完整覆盖，缺失项会导致构建失败。
- **构建产物一致性**：Vite 输出固定到 `dist/assets/[name]-[hash].*`，Go 侧通过 `embed.go` 静态嵌入，确保运行时路径稳定。
- **沙箱开发约束**：dev server 固定端口 3015、`strictPort: true`、HMR 默认关闭，避免沙箱 iframe 下热更新引发整页重载放大 transform error。

## 5. 适用边界

- 本卡片覆盖 `ui/`（Vivy Web UI）与 `studio/dsh-vivy-studio/`（DSH 主题包）两个前端子工程。
- `ui/agent-diva-source/` 为只读参考源码，不属于本仓库实现，不纳入样式体系描述。
- 后端 Go 代码不涉及前端样式逻辑，仅负责嵌入与路由代理。