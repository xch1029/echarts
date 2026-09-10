---
kind: configuration_system
name: ECharts 配置系统：选项合并、主题、国际化与媒体查询
category: configuration_system
scope:
    - '**'
source_files:
    - src/core/echarts.ts
    - src/model/Global.ts
    - src/model/OptionManager.ts
    - src/model/globalDefault.ts
    - src/core/locale.ts
    - src/theme/dark.ts
    - src/util/types.ts
    - src/preprocessor/backwardCompat.ts
    - src/i18n/langEN.ts
    - src/i18n/langZH.ts
---

## 1. 系统与架构概览

ECharts 的配置系统围绕 `setOption` 这一核心 API 构建，采用「默认值 → 主题 → 用户选项 → 组件默认值」的多层合并模型。入口在 `src/core/echarts.ts` 的 `ECharts.setOption`，内部委托给 `src/model/Global.ts` 的 `GlobalModel`，再由 `src/model/OptionManager.ts` 解析原始选项（支持 `baseOption` / `timeline` / `media`），最终通过拓扑排序对每个组件调用 `mergeOption`。

- **默认配置**：`src/model/globalDefault.ts` 提供全局默认值（颜色、动画阈值、字体、`useUTC`、`progressiveThreshold` 等）。
- **主题系统**：`src/theme/dark.ts` 是内置暗色主题；`ECharts._updateTheme` 支持传入主题名或主题对象，并通过 `GlobalModel.setTheme` 触发重建。
- **国际化**：`src/core/locale.ts` 维护 `localeStorage`，注册 `ZH`/`EN` 并可通过 `registerLocale` 扩展；初始化时根据 `document.documentElement.lang` / `navigator.language` 推导 `SYSTEM_LANG`，构造 `LocaleOption`。
- **媒体查询**：`OptionManager.getMediaOption` 基于当前宽高匹配 `media.query`（支持 `width`/`height`/`aspectRatio`，前缀 `min`/`max`），按声明顺序叠加生效。
- **时间线**：`OptionManager.getTimelineOption` 从 `timelineOptions` 中按当前索引取配置并合并。

## 2. 关键文件与职责

| 文件 | 职责 |
|---|---|
| `src/core/echarts.ts` | `ECharts` 类：接收 `setOption`/`setTheme`，创建 `GlobalModel`、`OptionManager`，驱动渲染周期 |
| `src/model/Global.ts` | `GlobalModel`：维护 `_theme`、`_locale`、组件映射；实现 `_mergeOption` 的拓扑合并与 `restoreData` |
| `src/model/OptionManager.ts` | 解析 `rawOption`，拆分 `baseOption`/`timelineOptions`/`mediaList`/`mediaDefault`，管理媒体查询匹配 |
| `src/model/globalDefault.ts` | 全局默认配置常量（颜色、动画、渐进渲染阈值等） |
| `src/core/locale.ts` | 语言包注册、默认语言推导、`createLocaleObject` 合并逻辑 |
| `src/theme/dark.ts` | 内置暗色主题对象（覆盖 `color`、`backgroundColor`、各组件样式） |
| `src/util/types.ts` | `ECBasicOption`、`ECUnitOption`、`MediaUnit`、`ThemeOption` 等类型定义 |
| `src/preprocessor/backwardCompat.ts` | 旧版 option 向后兼容预处理钩子 |
| `i18n/lang*.ts` | 各语言文案模块，被 `locale.ts` 默认导入 |

## 3. 配置加载与合并流程

1. **入口**：`ECharts.setOption(option, opts)` 校验不在主循环内，必要时新建 `OptionManager` 和 `GlobalModel`。
2. **预处理**：`OptionManager.setOption` 先克隆 rawOption，再调用注册的 `optionPreprocessorFuncs`（如 `backwardCompat`）。
3. **解析**：`parseRawOption` 识别 `baseOption`、根级 `timeline`/`options`/`media`，将 media 条目拆为 `mediaList` 与首个无 `query` 的 `mediaDefault`。
4. **挂载**：`mountOption` 返回用于首次初始化的 baseOption（recreate 时使用备份）。
5. **合并**：`GlobalModel._resetOption` 依次合并 baseOption → timelineOption → mediaOptions；非组件字段直接 `merge`，组件字段经 `ComponentModel.topologicalTravel` 拓扑顺序处理。
6. **默认值注入**：`initBase` 中先 `mergeTheme(baseOption, theme.option)`，再 `merge(baseOption, globalDefault, false)`，最后才进入组件 merge。
7. **运行时更新**：`resetOption('media' | 'timeline')` 仅恢复数据并按需重新合并对应片段。

## 4. 约定与约束

- **选项结构约定**：
  - 未声明 `baseOption` 时，根 option 即作为 baseOption，但 `timeline`/`options`/`media` 会被剥离到对应位置（见 `parseRawOption` 注释中的 `[RAW_OPTION_PATTERNS]`）。
  - 声明了 `baseOption` 后，除 ec2 兼容的根级 `timeline` 外，其他属性必须放在 `baseOption` 内。
- **媒体查询优先级**：后声明的 media 覆盖先声明的；若无任何匹配且存在 `mediaDefault`，则回退到它。
- **时间线不合并**：多次 `setOption` 若都含 `timeline.options`，后者直接替换前者，不做合并。
- **组件缺失诊断**：`checkMissingComponents` 在 dev 模式下打印缺失组件的 import 提示。
- **禁止循环调用**：`IN_EC_CYCLE_KEY` 标记下禁止再次调用 `setOption`，防止嵌套更新。
- **主题切换**：`setTheme` 会 clone 主题对象、执行 backwardCompat，然后调用 `ecModel.setTheme` 触发重建。
- **国际化**：`registerLocale` 以大写 key 存储；自定义 locale 会与默认 EN locale 合并（clone + merge），保证缺失键有兜底。
- **默认值不可变**：`globalDefault` 是静态导出对象，合并时使用 `merge(..., false)` 避免污染原对象。
- **ARIA 自动启用**：当 `baseOption.aria` 存在但未显式设置 `enabled` 时，自动设为 `true`。

## 5. 扩展点

- **OptionPreprocessor**：通过 `registerPreprocessor`（在 `core/echarts.ts` 中维护的 `dataProcessorFuncs` 类似机制）可插入预处理函数，修改原始 option。
- **自定义主题**：传入字符串名称时由 `themeStorage` 查找，或直接传主题对象。
- **自定义语言**：调用 `registerLocale(code, obj)` 注册新语言，再通过 `chart.setOption({ locale: code })` 使用。
- **replaceMerge**：`setOption` 的 `opts.replaceMerge` 允许指定 mainType 进行“替换合并”，清空该类型其余组件后再合并。

## 6. 适用性说明

本仓库是一个前端可视化库，其“配置系统”并非传统意义上的应用配置文件（如 `.env`、`.yaml`），而是以 JavaScript/TypeScript 对象形式通过 `setOption` 动态装配图表行为、外观与本地化。上述分析完整覆盖了该库如何加载、分层、合并与切换运行时配置的全部实现。