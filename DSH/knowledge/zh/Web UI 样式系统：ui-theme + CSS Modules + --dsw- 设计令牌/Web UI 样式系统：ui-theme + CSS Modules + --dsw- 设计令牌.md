---
kind: frontend_style
name: Web UI 样式系统：ui-theme + CSS Modules + --dsw-* 设计令牌
category: frontend_style
scope:
    - '**'
source_files:
    - docs/web-styling.md
    - docs/web-styling.zh.md
    - packages/client/tsdown.client.ts
    - packages/client/ui-theme/package.json
    - packages/client/AGENTS.md
    - apps/web/tests/lifecycle-chrome.e2e.ts
    - apps/web/tests/settings-chrome.e2e.ts
    - .agents/notes/implemented/bug-fix/2026-07-28-themed-scrollbars-and-reserved-gutter.md
---

## 1. 采用的体系
DeepSeek Harness 的浏览器客户端采用 **插件化 Cordis 架构** 下的分层样式系统，核心由三个包协作完成：
- `@deepseek-ai/dsh-client-ui-theme`：定义并注入全局 `--dsw-*` 静态色阶、语义别名（`--dsw-alias-*`）、排版、动效、渐变、阴影、滚动条样式以及明/暗主题偏好。它通过 Cordis 的 `dsh.client` 机制在应用启动时立即注入。
- `@deepseek-ai/dsh-client-ui-layout`：将 ui-theme 解析出的主题快照应用到文档（设置 `body` 属性与 token），是主题的实际“渲染者”。
- 各功能 UI 包（`ui-conversation`、`ui-sidebar`、`ui-tool`、`ui-primitives`、`ui-settings-plugins` 等）：消费语义别名，不自行定义全局主题。

构建侧使用 **tsdown + lightningcss** 自定义管线：`.module.css` 编译为带哈希 class 映射的 CSS Modules；普通 `.css` 作为全局样式被注入到 `<style data-plugin-css="...">` 标签中；`.css?inline` 导出编译后的文本供生命周期 effect 注入。该管线位于 `packages/client/tsdown.client.ts`，明确禁止使用 tsdown 内置 CSS 管道，改用虚拟模块前缀 `\0dsh-css:` / `\0dsh-global-css:` / `\0dsh-inline-css:` 绕过。

## 2. 关键文件与包
- 样式规范文档：`docs/web-styling.md`（及 `.zh.md`）——权威的风格所有权与组件规则。
- 构建管线：`packages/client/tsdown.client.ts` ——CSS Modules、全局样式、内联样式的编译与注入逻辑。
- 主题包：`packages/client/ui-theme/`（package.json 声明 `dsh.client.immediately: true`，依赖 `clsx` 与 `@deepseek-ai/schemastery`）。
- 布局包：`packages/client/ui-layout/` ——负责把主题快照落到 DOM。
- 功能 UI 包：`packages/client/ui-conversation`、`ui-sidebar`、`ui-tool`、`ui-primitives`、`ui-settings-plugins`、`ui-agent-preset`、`ui-approval`、`ui-attachment`、`ui-brand-official`、`ui-chat` 等，均通过 workspace 依赖消费上述基础包。
- 测试断言：`apps/web/tests/lifecycle-chrome.e2e.ts`、`settings-chrome.e2e.ts`、`navigation-panes.e2e.ts` 等通过 `getComputedStyle(document.body).getPropertyValue('--dsw-alias-*')` 校验 token 是否生效。

## 3. 架构与约定
- **职责分离**：全局样式表归 `ui-theme/src/styles/`；组件样式以 CSS Modules 形式放在组件同级目录。组件可定义局部 CSS 自定义属性，但共享的颜色、排版、层级、动效必须来自主题包。
- **令牌体系**：所有颜色、间距、字号等通过 `--dsw-*` 静态 scale 与 `--dsw-alias-*` 语义别名引用，禁止在功能组件中直接写字面量颜色。
- **主题切换**：明/暗偏好由 ui-theme 管理，ui-layout 根据偏好设置 body 属性（如 `data-ds-dark-theme`），再通过 CSS 选择器覆盖 `--dsw-*` 值。
- **滚动条统一**：滚动条样式集中在 `ui-theme/src/styles/scrollbar.css`，抬升表面需成对重新绑定 `--dsh-scrollbar-thumb` / `--dsh-scrollbar-thumb-hover`，否则 hover 状态会回退到基础层。
- **无第三方 UI 框架**：明确不使用 Tailwind、styled-components、Emotion 等组件库或原子 CSS 框架，仅用原生 CSS + CSS Modules + clsx。
- **源码语言**：代码注释使用英文（见 `packages/client/AGENTS.md`）。

## 4. 约束与强制规则
以下规则来自文档与构建管线的显式约定，具有工程级约束力：
- **禁止引入组件库或 Tailwind**：`docs/web-styling.md` 明文规定 “Use CSS Modules and `clsx`; do not add a component library or Tailwind.”
- **禁止在功能组件中定义全局主题**：feature packages 只能消费语义别名，不得再定义另一个全局主题。
- **禁止在组件 CSS 中使用主题选择器**：light/dark 覆盖必须由主题所有者维护。
- **字体大小必须搭配行高**，且应复用主题排版变量。
- **保留键盘焦点可见性与 reduced-motion 行为**，添加过渡或 hover-only 控件时必须遵守。
- **源码文本、终端输出、diff 行在需要列对齐的组件中保持不换行**，并使用共享滚动条样式而非组件内自定义滚动条选择器。
- **演示逻辑放入 CSS**：React inline style 仅允许传递组件局部自定义属性值，不得编码主题分支。
- **构建阶段强制**：`tsdown.client.ts` 通过虚拟模块前缀和 `lightningcss` 转换，确保每个插件包的样式以独立 `<style data-plugin="...">` 注入，避免跨插件样式泄漏；相对 `.css` 导入保持相对路径以便 Vite 管理 class 哈希。
- **令牌变更流程**：修改共享 token 必须在 `ui-theme` 中进行，再由功能包消费其语义别名；公共样式契约变更时需更新 owning package reference（`docs/web-styling.md` 第 25 行）。
- **滚动条重绑定契约**：任何既滚动又绘制抬升表面的样式表都必须成对重新绑定 `--dsh-scrollbar-thumb{,-hover}`，该契约由 `ui-theme/src/styles/scrollbar.css` 与对应测试共同保障（参见 `.agents/notes/implemented/bug-fix/2026-07-28-themed-scrollbars-and-reserved-gutter.md`）。