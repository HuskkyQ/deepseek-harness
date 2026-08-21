# Theme Accent Plugin (ui-accent) Design

English | [中文](2026-08-15-theme-accent-plugin-design.zh.md)

Date: 2026-08-15 · Status: Approved by the user; implementation pending

## 1. Background and goals

The DeepSeek Harness Web client already has a theme system: `ui-theme` provides `--dsw-*` tokens (static color scales + semantic aliases) and light/dark/system preferences, while `ui-layout` applies the parsed snapshot to the document. The requested addition is a “theme accent” capability: under **General Settings → Appearance**, the user selects a brand primary color (accent) inline; the selected color affects brand emphasis positions such as primary buttons, and the selection persists across refreshes.

Goals:

- Create an independent client plugin package `@deepseek-ai/dsh-client-ui-accent` (`packages/client/ui-accent`), following the “one UI feature = one plugin package” directory convention
- Embed the palette inline in the AppearanceRow (the `appearance` entry of `settings.general.item`), instead of adding a new general settings row
- Cover only the brand primary color and its button-derived tokens (the scope confirmed by the user)
- Persist the preference through Host settings (`$DSH_HOME/settings.yaml`), retaining it after refresh

## 2. Confirmed decisions (approved by the user)

| Decision | Conclusion |
| --- | --- |
| Plugin form | Independent package + preset palette + Host settings persistence |
| Scope | Brand primary only (`--dsw-alias-brand-*` and derived button primary colors); does not cover business emphasis colors, sidebar accent, or dimmed states |
| Presets | 5 colors: default (original neutral) + DeepSeek blue + violet + emerald green + amber |
| UI location | Embedded inline in the General Settings → Appearance row (the user's final addition) |

## 3. Option comparison (record)

- **A (selected)**: Independent package + `theme.overrideTokens()` overlay + new settings namespace. Reuses all existing project patterns; the disposer restores automatically, and the theme registry is not polluted.
- B: Register each preset as a complete “theme” with `theme.register()`. This conflicts with the light/dark/system three-mode UI, and third-party registered themes are “process-only and not persistent,” which conflicts with the persistence goal. Rejected.
- C: Persist through client `localStorage`. This breaks the settings single source of truth and conflicts with the host-backed preference decision. Rejected.

## 4. Architecture and data flow

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

Key properties:

- The overlay source is fixed to `'@deepseek-ai/dsh-client-ui-accent'`; changing presets replaces the entire layer under the same source (the existing ThemeRuntime semantics); selecting `default` releases the layer and fully falls back to the original version
- The overlay is decoupled from the theme; each token carries both light and dark values, so it follows light/dark switching automatically
- `ui-theme` only declares the empty slot; `ui-accent` has zero knowledge of it, and the dependency direction remains layered downward

## 5. Package structure and file list

New package `packages/client/ui-accent` (mirroring the `ui-theme` structure):

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

### 5.1 Settings contract (`theme-accent-settings.ts`)

- Namespace: `ui-accent`; field: `accent`
- `ACCENT_PRESET_IDS = ['default', 'blue', 'violet', 'green', 'amber']`
- Schema: `z.object({ accent: z.union([...ACCENT_PRESET_IDS]).default('default') })`
- Host-side `apply`: `ctx.inject(['settings'], (c) => c.settings.register(settingsNamespace('ui-accent'), AccentSettingsSchema))`; no boot injection (see Section 9 for the limitation)

### 5.2 Preset token table (`presets.ts`)

Each non-default preset overrides four tokens with both light and dark values (the `ThemeTokenOverrides` form):

| Token | Purpose | Description |
| --- | --- | --- |
| `--dsw-alias-brand-primary` | Brand primary color | Primary button fill cascades through `button-primary-fill: var(--dsw-alias-brand-primary)` |
| `--dsw-alias-brand-text` | Brand emphasis text | It currently has no consumer; it is a brand contract token and is overridden together with the primary color |
| `--dsw-alias-brand-primary-invert` | Text color on the primary color | Light primary colors (green/amber) use dark text to preserve contrast |
| `--dsw-alias-button-primary-hover` | Primary button hover | It is currently a static neutral color; without overriding it, the color changes abruptly on hover |

Initial preset values (light / dark; verify contrast for both palettes during implementation, with numerical adjustments allowed):

| Preset | brand-primary / brand-text | invert | hover |
| --- | --- | --- | --- |
| blue (DeepSeek blue) | `rgb(65,118,230)` / `rgb(86,134,254)` | `rgb(255,255,255)` in both modes | `rgb(37,99,235)` / `rgb(103,158,254)` |
| violet | `rgb(124,92,252)` / `rgb(139,124,255)` | `rgb(255,255,255)` in both modes | `rgb(109,72,247)` / `rgb(156,144,255)` |
| green (emerald green) | `rgb(34,197,94)` / `rgb(78,209,126)` | `rgb(35,60,44)` in both modes (green-900 dark text) | `rgb(22,163,74)` / `rgb(134,239,172)` |
| amber | `rgb(245,158,11)` / `rgb(247,173,49)` | `rgb(39,36,31)` in both modes (amber-900 dark text) | `rgb(217,119,6)` / `rgb(251,191,36)` |

`default`: no overlay is applied; the original neutral brand colors are used completely.

## 6. Inline integration in the Appearance row (slot composition)

- Add `children: { 'settings.general.appearance.accent': { kind: 'single', scope: 'root' } }` to the `AppearanceRow` registration (children = declaration + rendering authorization)
- Add `PropsRenderSlots<'settings.general.appearance.accent'>` to the `AppearanceRow` component props, and render `renderSlot('settings.general.appearance.accent', {})` after the cube row
- Add the new slot's SlotMap type to `packages/client/ui-settings/src/client/contract/slots.ts` (the base layer of the settings domain, alongside `'settings.general.item'`; the owner has empty props, following the empty-props convention of `SettingsGeneralItemOwnerProps`)
- `ui-accent` uses order-independent pattern registration: `ctx.slots.inject('settings.general.appearance.accent', () => ctx.slots.register({ name: 'settings.general.appearance.accent', store, locale, inject }, AccentRow))` (the same pattern as locale → Language and ui-theme → Appearance)
- AccentRow visual: group label “主题色” (en: Accent color) + 5 circular swatches; the default swatch renders the original neutral color with an outline indicating “not enabled”; selected state uses a ring; semantic `aria-pressed`

Dependency direction: `ui-theme` (owner of all Appearance row slots) does not depend on `ui-accent`; `ui-accent` depends on the `ctx.theme` service (type-only import from ui-theme/client, as with cordis-client-runner), `settingsScope`, and slots.

## 7. Persistence and apiproxy allowlist

- Add `'ui-accent'` to `packages/host/apiproxy/src/api-proxy.ts`'s `WEB_SETTINGS_NAMESPACES` (the comment there explicitly states that “exposing a namespace is decided here”)
- Update the exposed-name assertion in `packages/host/apiproxy/tests/api-proxy-config.spec.ts` and the allowlist description in its README (md + zh)

## 8. Composition integration and build wiring

- In `packages/bundle/web-app/cordis.patch.yml`, add `- id: ui-accent` / `name: '@deepseek-ai/dsh-client-ui-accent'` after `ui-theme` in the client tree
- Add a path mapping to `tsconfig.base.json`; add a project reference to `tsconfig.client.json`; add a package entry to `knip.json`
- `package.json`: `dsh.client.inject = [ui-theme, connection, runtime, locale, ui-settings, api-remotes]`, with matching peerDeps + `ui-slots` (Props type); **do not depend on** `ui-primitives` (the swatches are pure CSS circles and need no icons) or `ui-settings-general` (the slot type belongs at the ui-settings contract layer)

## 9. Boundaries and limitations

- **On refresh**, the default brand color renders first; after the client plugin activates, the selected theme color is applied. No host-side boot injection is used (reserved for a later extension and must be documented in the README)
- **If ui-accent is omitted from the composition**, the Appearance row's empty slot renders nothing; verify the empty single-slot behavior during implementation and document it in the README
- Do not override dimmed states such as `button-primary-dimmed`, `state-business-primary`, or the sidebar accent (scope decision)
- The overlay affects only the semantic `--dsw-alias-*` layer and does not add a global stylesheet

## 10. Tests and verification

- `tests/presets.spec.ts`: preset-table completeness (all 4 non-default presets × 4 tokens × both light/dark values; the union of IDs matches the settings schema; `default` has no tokens)
- `tests/apply.client.spec.ts`: settings change → correct `overrideTokens` arguments (fixed source and correct tokens); calling again with the same source replaces the layer; selecting `default` releases the layer (disposer called); slot registration arguments (name/store/inject/locale)
- `tests/AccentRow.client.spec.tsx` (jsdom pragma): render 5 swatches, `aria-pressed` reflects store state, clicking invokes `setAccent(id)`, and the zh copy is “主题色”
- apiproxy spec: exposed-name allowlist contains `ui-accent`
- e2e (`apps/web/tests`, settings-chrome mode): click the violet swatch → the body's inline `--dsw-alias-brand-primary` equals the preset value → it persists after refresh → clicking default restores the original value
- 100% coverage gate for the client package; run the repository check ladder (lint / typecheck / coverage)
- Build the Web artifact and verify it after refreshing the existing GUI address (port 3080 in this session)

## 11. Implementation order overview

1. New package skeleton: manifest/tsconfig/tsdown + settings contract + preset table
2. Host-side settings registration; client-side apply (scope binding, overlay lifecycle)
3. SlotMap type + the ui-theme AppearanceRow empty slot + AccentRow UI (store/i18n)
4. apiproxy allowlist, cordis.patch.yml, tsconfig/knip wiring
5. Tests (unit tests + apiproxy spec + e2e)
6. Documentation (README zh/en, apiproxy README, and config-catalog if it is generated output; then run the generator)
7. Full verification ladder + hands-on Web GUI verification

## 12. Change list

Added: `packages/client/ui-accent/` (all files), `docs/superpowers/specs/2026-08-15-theme-accent-plugin-design.md`

Modified: `packages/client/ui-settings/src/client/contract/slots.ts`, `packages/client/ui-theme/src/client/index.ts`, `packages/client/ui-theme/src/client/AppearanceRow.tsx`, `packages/host/apiproxy/src/api-proxy.ts` (+tests+README), `packages/bundle/web-app/cordis.patch.yml`, `tsconfig.base.json`, `tsconfig.client.json`, `knip.json`
