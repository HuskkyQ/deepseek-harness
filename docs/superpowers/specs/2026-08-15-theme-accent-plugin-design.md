# 主题色插件（ui-accent）设计

日期：2026-08-15 · 状态：已获用户批准，待实现

## 1. 背景与目标

DeepSeek Harness 的 Web 客户端已有一套主题系统：`ui-theme` 提供 `--dsw-*` token（静态色阶 + 语义别名）与 light/dark/system 偏好，`ui-layout` 负责把解析后的快照应用到文档。用户希望增加“主题色”能力：在 **通用设置 → 外观** 行内选择一套品牌主色（accent），选中的颜色影响按钮主色等品牌强调位，选择跨刷新持久化。

目标：

- 新建独立客户端插件包 `@deepseek-ai/dsh-client-ui-accent`（`packages/client/ui-accent`），遵循“一个 UI 功能 = 一个插件包”的目录制度
- 在外观行（AppearanceRow，`settings.general.item` 的 `appearance` 条目）内嵌色板，而不是新增通用设置行
- 只覆盖品牌主色及其按钮派生 token（经用户确认的范围）
- 偏好经 Host settings 持久化（`$DSH_HOME/settings.yaml`），刷新后保留

## 2. 已确认的决策（用户拍板）

| 决策点 | 结论 |
| --- | --- |
| 插件形态 | 独立插件包 + 预设色板 + Host settings 持久化 |
| 生效范围 | 仅品牌主色（`--dsw-alias-brand-*` 及按钮主色派生），不覆盖业务强调色、侧边栏 accent、dimmed 态 |
| 预设 | 5 色：默认（原版中性）+ DeepSeek 蓝 + 紫罗兰 + 翠绿 + 琥珀 |
| UI 位置 | 通用设置 → 外观行内嵌（用户最后补充的要求） |

## 3. 方案对比（记录）

- **A（选定）**：独立包 + `theme.overrideTokens()` 覆盖层 + 新 settings namespace。复用原项目全部范式，disposer 自动还原，不污染主题注册表。
- B：把每个预设注册成完整“主题”（`theme.register()`）。与 light/dark/system 三档 UI 冲突，且第三方注册主题“仅进程内、不持久化”，与目标冲突。弃。
- C：客户端 localStorage 持久化。破坏 settings 单一事实源，与 host-backed 偏好决策相悖。弃。

## 4. 架构与数据流

```
设置页外观行色板点击
  → AccentRow injected 回调 setAccent(id)
  → settingsScope.update({ accent: id })（Host settings API）
  → 落盘 $DSH_HOME/settings.yaml，回推浏览器（settings 变更推送）
  → ui-accent apply 世界的 scope 订阅被触发
  → dispose 旧覆盖层；id ≠ 'default' 时 theme.overrideTokens(source, preset.tokens)
  → ThemeRuntime 重组快照并发布 theme/change
  → ui-layout presenter 应用内联 alias token → 界面换色
```

关键性质：

- 覆盖层 source 固定为 `'@deepseek-ai/dsh-client-ui-accent'`；换预设 = 同一 source 替换整个层（ThemeRuntime 既有语义），选 `default` 时释放层、完全回落原版
- 覆盖层与主题解耦，每 token 携带 light/dark 双值，明暗切换自动跟随
- ui-theme 只声明空洞，对 ui-accent 零知识；依赖方向保持一层层向下

## 5. 包结构与文件清单

新包 `packages/client/ui-accent`（镜像 `ui-theme` 结构）：

```
packages/client/ui-accent/
  package.json                 # dsh.client.inject 含 ui-theme；platform: web
  tsconfig.json                # 引用：ui-theme、ui-settings、ui-slots、ui-locale、ui-runtime、
                               #        host/webserver? 不需要；settings/settings、vendor/cordis
  tsdown.config.ts
  README.md / README.zh.md / README.i18n.yaml
  src/
    theme-accent-settings.ts   # namespace、AccentPresetId、schema（host/client 共享）
    presets.ts                 # 4 个非默认预设的 token 表（light/dark 双值）
    index.ts                   # host 侧：ctx.inject(['settings']) 注册 namespace
    client/
      index.ts                 # browser 侧 apply：scope 绑定 + 覆盖层生命周期 + slot 注册
      AccentRow.tsx            # 小组标签「主题色」+ 5 个圆形色板
      AccentRow.module.css
      settings-store.ts        # createAccentRowStore()：{ accent, revision } + sync action
      locales.ts               # zh / en
  tests/
    presets.spec.ts
    apply.client.spec.ts
    AccentRow.client.spec.tsx
```

### 5.1 settings 契约（`src/theme-accent-settings.ts`）

- namespace：`ui-accent`，字段 `accent`
- `ACCENT_PRESET_IDS = ['default', 'blue', 'violet', 'green', 'amber']`
- schema：`z.object({ accent: z.union([...ACCENT_PRESET_IDS]).default('default') })`
- host 侧 `apply`：`ctx.inject(['settings'], (c) => c.settings.register(settingsNamespace('ui-accent'), AccentSettingsSchema))`；不做 boot 注入（见 §9 限制）

### 5.2 预设 token 表（`src/presets.ts`）

每个非默认预设覆盖 4 个 token 的 light+dark 双值（`ThemeTokenOverrides` 形态）：

| token | 作用 | 说明 |
| --- | --- | --- |
| `--dsw-alias-brand-primary` | 品牌主色 | 主按钮填充经 `button-primary-fill: var(--dsw-alias-brand-primary)` 级联 |
| `--dsw-alias-brand-text` | 品牌强调文字 | 目前无消费者，属品牌契约 token，随主色一起覆盖 |
| `--dsw-alias-brand-primary-invert` | 主色上的文字色 | 浅色主色（绿/琥珀）用深字保证对比度 |
| `--dsw-alias-button-primary-hover` | 主按钮 hover | 现为静态中性色，不覆盖会跳色 |

预设初值（light / dark，实现时逐一对两套配色核对对比度，允许微调数值）：

| 预设 | brand-primary / brand-text | invert | hover |
| --- | --- | --- | --- |
| blue（DeepSeek 蓝） | `rgb(65,118,230)` / `rgb(86,134,254)` | `rgb(255,255,255)` 双模式 | `rgb(37,99,235)` / `rgb(103,158,254)` |
| violet（紫罗兰） | `rgb(124,92,252)` / `rgb(139,124,255)` | `rgb(255,255,255)` 双模式 | `rgb(109,72,247)` / `rgb(156,144,255)` |
| green（翠绿） | `rgb(34,197,94)` / `rgb(78,209,126)` | `rgb(35,60,44)` 双模式（green-900 深字） | `rgb(22,163,74)` / `rgb(134,239,172)` |
| amber（琥珀） | `rgb(245,158,11)` / `rgb(247,173,49)` | `rgb(39,36,31)` 双模式（amber-900 深字） | `rgb(217,119,6)` / `rgb(251,191,36)` |

`default`：不叠加覆盖层，完全走原版中性品牌色。

## 6. 外观行内嵌（slot 组合）

- `AppearanceRow` 的 register 增加 `children: { 'settings.general.appearance.accent': { kind: 'single', scope: 'root' } }`（children = 声明 + 授权渲染）
- `AppearanceRow` 组件 props 增加 `PropsRenderSlots<'settings.general.appearance.accent'>`，在立方体行后渲染 `renderSlot('settings.general.appearance.accent', {})`
- 新 slot 的 SlotMap 类型加在 `packages/client/ui-settings/src/client/contract/slots.ts`（settings 域基础层，与 `'settings.general.item'` 同处；owner 为空 props，同 `SettingsGeneralItemOwnerProps` 的空 props 惯例）
- `ui-accent` 用顺序无关的模式注册：`ctx.slots.inject('settings.general.appearance.accent', () => ctx.slots.register({ name: 'settings.general.appearance.accent', store, locale, inject }, AccentRow))`（与 locale→Language、ui-theme→Appearance 同款）
- AccentRow 视觉：小组标签「主题色」（en: Accent color）+ 5 个圆形色板；默认色板渲染原版中性色并带描边示意“未启用”；选中态圆环；`aria-pressed` 语义

依赖方向：ui-theme（外观行所有者）不依赖 ui-accent；ui-accent 依赖 `ctx.theme` 服务（type-only import ui-theme/client，与 cordis-client-runner 同法）、`settingsScope`、slots。

## 7. 持久化与 apiproxy 白名单

- `packages/host/apiproxy/src/api-proxy.ts` 的 `WEB_SETTINGS_NAMESPACES` 增加 `'ui-accent'`（该处注释明确“暴露一个 namespace 是在这里做的决定”）
- 同步更新 `packages/host/apiproxy/tests/api-proxy-config.spec.ts` 的暴露名单断言与 README（md + zh）白名单说明

## 8. 组合接入与构建接线

- `packages/bundle/web-app/cordis.patch.yml`：client 树 `ui-theme` 之后加一行 `- id: ui-accent` / `name: '@deepseek-ai/dsh-client-ui-accent'`
- `tsconfig.base.json` 加路径映射；`tsconfig.client.json` 加 project reference；`knip.json` 加包条目
- `package.json`：`dsh.client.inject = [ui-theme, connection, runtime, locale, ui-settings, api-remotes]`，peerDeps 对应 + `ui-slots`（Props 类型）；**不依赖** `ui-primitives`（色板是纯 CSS 圆点，无需图标）与 `ui-settings-general`（slot 类型在 ui-settings 契约层）

## 9. 边界与限制

- **刷新瞬间**先渲染默认品牌色，客户端插件激活后应用所选主题色；不做 host 侧 boot 注入（留作后续扩展，需在 README 说明）
- 裁掉 ui-accent 的组合：外观行空洞为空——实现时验证空 single slot 的渲染行为并写入 README
- `button-primary-dimmed` 等 dimmed 态、`state-business-primary`、侧边栏 accent 不覆盖（范围决策）
- 覆盖层只影响 `--dsw-alias-*` 语义层，不新增全局样式表

## 10. 测试与验证

- `tests/presets.spec.ts`：预设表完整性（4 非默认预设 × 4 token × light/dark 双值齐全；id 与 settings schema 并集一致；default 无 tokens）
- `tests/apply.client.spec.ts`：settings 变更 → `overrideTokens` 入参正确（source 固定、tokens 正确）；同 source 再调用为替换；选 default 释放层（disposer 被调用）；slot 注册参数（name/store/inject/locale）
- `tests/AccentRow.client.spec.tsx`（jsdom pragma）：渲染 5 个色板、`aria-pressed` 反映 store 状态、点击回调 `setAccent(id)`、zh 文案「主题色」
- apiproxy spec：暴露名单含 `ui-accent`
- e2e（apps/web/tests，settings-chrome 模式）：点紫色板 → body 内联 `--dsw-alias-brand-primary` 为预设值 → 刷新后保留 → 点默认还原
- 客户端包 100% 覆盖率门禁；跑仓库 check ladder（lint / typecheck / coverage）
- 构建 Web 产物并在既有 GUI 地址刷新验证（本会话的 Web 端口 3080）

## 11. 实施顺序概览

1. 新包骨架：manifest/tsconfig/tsdown + settings 契约 + 预设表
2. host 侧 settings 注册；client 侧 apply（scope 绑定、覆盖层生命周期）
3. SlotMap 类型 + ui-theme AppearanceRow 空洞 + AccentRow UI（store/i18n）
4. apiproxy 白名单、cordis.patch.yml、tsconfig/knip 接线
5. 测试（单测 + apiproxy spec + e2e）
6. 文档（README zh/en、apiproxy README、config-catalog 若为生成产物则跑生成脚本）
7. 全量验证阶梯 + Web GUI 实测

## 12. 变更清单

新增：`packages/client/ui-accent/`（全部）、`docs/superpowers/specs/2026-08-15-theme-accent-plugin-design.md`

修改：`packages/client/ui-settings/src/client/contract/slots.ts`、`packages/client/ui-theme/src/client/index.ts`、`packages/client/ui-theme/src/client/AppearanceRow.tsx`、`packages/host/apiproxy/src/api-proxy.ts`（+tests+README）、`packages/bundle/web-app/cordis.patch.yml`、`tsconfig.base.json`、`tsconfig.client.json`、`knip.json`
