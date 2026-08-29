---
kind: frontend_style
name: Agent Diva GUI 前端样式体系：Tailwind + CSS 变量主题系统
category: frontend_style
scope:
    - '**'
source_files:
    - agent-diva-gui/src/styles.css
    - agent-diva-gui/tailwind.config.js
    - agent-diva-gui/postcss.config.js
    - agent-diva-gui/package.json
    - agent-diva-gui/src/composables/useTheme.ts
    - agent-diva-gui/src/App.vue
    - agent-diva-gui/src/components/settings/
    - agent-diva-gui/avatar-runtime-vrm/
    - agent-diva-gui/shared-avatar-protocol/
---

## 1. 使用的系统与工具

- **框架与构建**：基于 Vue 3 + TypeScript，使用 Vite 作为开发/构建工具（`vite.config.ts`），通过 `@vitejs/plugin-vue` 集成。
- **CSS 处理管线**：PostCSS + Tailwind CSS 3（`postcss.config.js` 启用 `tailwindcss`、`autoprefixer`），在 `src/styles.css` 顶部以 `@tailwind base/components/utilities` 引入。
- **桌面打包**：Tauri 2（`@tauri-apps/api ~2.11.1`）将 Vue SPA 打包为跨平台桌面应用；VRM 虚拟人运行时位于子目录 `avatar-runtime-vrm/`，共享协议位于 `shared-avatar-protocol/`。
- **图标库**：`@lucide/vue` 提供统一 SVG 图标，通过 CSS 变量控制尺寸与描边。
- **包管理器**：pnpm（`packageManager: pnpm@10.33.2`）。

## 2. 关键文件与位置

| 文件 | 作用 |
|---|---|
| `agent-diva-gui/src/styles.css` | 全局样式入口，定义全部 CSS 变量主题、组件层 `.app-shell/.sidebar/.nav-item` 等通用样式 |
| `agent-diva-gui/tailwind.config.js` | Tailwind 配置，扩展 `yandere`、`surface`、`ink`、`line`、`brand`、`accent`、`state` 语义色及 `tk-*` 字号/字重/圆角/阴影令牌 |
| `agent-diva-gui/postcss.config.js` | PostCSS 插件链（tailwindcss + autoprefixer） |
| `agent-diva-gui/package.json` | 依赖声明（Vue 3、Tailwind、Lucide、Tauri、Three.js/VRM 等） |
| `agent-diva-gui/src/composables/useTheme.ts` | 主题切换逻辑（读取/写入 `data-theme` 属性） |
| `agent-diva-gui/src/App.vue` | 根组件，挂载主题到 `<html>` / `<body>` |
| `agent-diva-gui/src/components/settings/*` | 设置面板中各主题选项的 UI |
| `agent-diva-gui/avatar-runtime-vrm/` | VRM 虚拟人运行时（独立 Vite 项目，含动画与模型资源） |
| `agent-diva-gui/shared-avatar-protocol/` | 前后端共享的 Avatar 命令/事件类型定义 |

## 3. 架构与设计约定

### 3.1 主题系统（CSS 变量 + data-theme 选择器）

所有视觉差异集中在 `styles.css` 中的四个主题块：
- `:root, :root[data-theme="love"]` — 默认粉白渐变主题
- `:root[data-theme="dark"]` — 深色主题（slate 底色 + 蓝色强调）
- `:root[data-theme="default"]` — 简约粉白主题
- `:root[data-theme="miku"]` — 初音未来青绿色主题

每个主题块覆盖同一组 CSS 变量，包括：
- 基础色：`--bg-app`、`--panel`、`--panel-solid`、`--line`、`--text`、`--text-muted`、`--shadow`、`--glass-blur`、`--radius`、`--radius-sm`
- 导航：`--nav-hover`、`--nav-active`、`--nav-active-border`、`--nav-icon`、`--nav-icon-active`、`--nav-icon-size`、`--nav-icon-stroke`、`--nav-icon-opacity`
- 气泡：`--bubble-user-radius`、`--bubble-assistant-radius`、`--bubble-shadow`、`--bubble-glow`、`--bubble-padding`、`--bubble-max-width`、`--bubble-user-bg`、`--bubble-assistant-bg`
- 强调色：`--accent`、`--accent-light`、`--accent-glow`、`--accent-border`、`--accent-bg-light`、`--accent-bg-hover`
- 语义状态：`--danger`/`--success`/`--warning`/`--info` 及其 `-bg` 变体
- 聊天区专用：`--chat-bg`、`--chat-text`、`--chat-bar-bg`、`--chat-input-bg`、`--chat-avatar-*`、`--subview-bg`、`--conv-sidebar-*`

主题无关的设计令牌集中在 `:root` 顶层：字号阶梯（`--font-size-xs..xl`）、行高、字重（`--font-weight-normal/medium/semibold`）、间距（`--space-1..6`）、身份色板、Mate 玻璃色板。

### 3.2 Tailwind 语义令牌映射

`tailwind.config.js` 将 CSS 变量映射为 Tailwind 扩展类名：
- 颜色：`colors.surface.*` → `var(--panel/solid/raised/sunken)`；`colors.ink.*` → `var(--text/muted/faint)`；`colors.line.*` → `var(--line/strong)`；`colors.brand.*`、`colors.accent.*`、`colors.state.*` 同理。
- 字体：`fontSize.tk-*` 与 `fontWeight.tk-*` 绑定 `--font-size-*` / `--font-weight-*`。
- 形状/阴影：`borderRadius.tk*`、`boxShadow.tk` 绑定 `--radius*`、`--shadow`。
- 品牌色阶：`yandere.50..900` 提供完整粉色阶梯。

组件中使用 `tk-*` 前缀的 Tailwind 类（如 `tk-base`、`tk-md`、`tk-sm`、`tk`）来引用设计令牌，避免与 Tailwind 默认的 `text-*`、`rounded-*` 冲突。

### 3.3 组件级样式组织

- 全局布局组件（`.app-shell`、`.sidebar`、`.nav-item`、`.nav-group-header`、`.popup-menu-item`、`.menu-toggle` 等）集中在 `styles.css` 的 `@layer components` 区块，按功能分节注释（Grid 布局、侧边栏、导航分组）。
- 业务组件（`ChatView`、`SettingsView`、`ApprovalCenterDrawer`、`MaskCard` 等）位于 `src/components/`，采用 Vue SFC 内联 `<style scoped>` 或组合式样式，复用上述全局令牌。
- 设置项（`src/components/settings/`）提供主题选择器等可配置 UI。

### 3.4 响应式策略

- 通过 CSS 变量 `--sidebar-width`（260px）与 `--sidebar-collapsed-width`（56px）驱动侧边栏展开/折叠，配合 `.app-shell.sidebar-expanded` 切换 grid-template-columns。
- Mate 沉浸式/专注模式通过 `.app-shell.mate-immersive`、`.app-shell.mate-focus-mode` 隐藏侧边栏列宽为 0。
- 未使用媒体查询做断点适配，主要依赖 Tauri 窗口尺寸变化 + 变量驱动的弹性布局。

## 4. 约定与约束

- **新增主题**：必须在 `styles.css` 中添加新的 `:root[data-theme="xxx"]` 块，覆盖全部必需变量（否则组件会回退到缺失变量的默认值）。
- **新增颜色令牌**：先在 `styles.css` 的某个主题块中定义 `--xxx` 变量，再在 `tailwind.config.js` 的 `theme.extend.colors` 中暴露为 Tailwind 类，最后组件用 `text-xxx` / `bg-xxx` 引用。
- **字号/字重/圆角/阴影**：统一使用 `tk-*` 前缀的 Tailwind 类（`tk-xs..tk-xl`、`tk-normal/medium/semibold`、`tk/tk-sm`、`tk`），禁止直接使用 `text-sm`、`rounded-md` 等 Tailwind 默认值，以保证主题切换时这些属性随主题变量变化。
- **主题切换**：通过 `useTheme` composable 修改 `document.documentElement` 的 `data-theme` 属性，新主题立即生效（无 JS 重渲染）。
- **图标尺寸/描边**：统一通过 `--nav-icon-size`、`--nav-icon-stroke`、`--nav-icon-opacity` 等变量控制，避免在组件中硬编码像素。
- **气泡风格**：用户/助手消息气泡统一使用 OpenAkita 风格的非对称圆角（`--bubble-user-radius`、`--bubble-assistant-radius`），由主题变量集中管理。
- **VRM/Mate 相关样式**：位于 `avatar-runtime-vrm/` 子项目中，与主 GUI 解耦，通过 `shared-avatar-protocol` 共享类型，不混入主应用的 `styles.css`。