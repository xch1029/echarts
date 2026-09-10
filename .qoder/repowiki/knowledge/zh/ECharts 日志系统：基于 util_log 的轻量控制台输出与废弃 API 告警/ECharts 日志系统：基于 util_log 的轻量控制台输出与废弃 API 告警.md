---
kind: logging_system
name: ECharts 日志系统：基于 util/log 的轻量控制台输出与废弃 API 告警
category: logging_system
scope:
    - '**'
source_files:
    - src/util/log.ts
    - src/util/styleCompat.ts
    - src/animation/customGraphicKeyframeAnimation.ts
    - src/animation/customGraphicTransition.ts
    - src/animation/universalTransition.ts
    - src/chart/bar/BarView.ts
    - src/chart/boxplot/boxplotTransform.ts
---

## 1. 使用的系统与方案

ECharts 没有引入第三方日志框架，而是内置了一个极简的、面向浏览器控制台的日志子系统，核心位于 `src/util/log.ts`。该模块提供 `log`、`warn`、`error`、`deprecateLog`、`deprecateReplaceLog`、`makePrintable`、`throwError` 等导出函数，统一通过 `console.log/warn/error` 输出，并在所有消息前加上 `[ECharts] ` 前缀（`ECHARTS_PREFIX`），便于在浏览器开发者工具中过滤。

- 环境开关：所有废弃 API 警告和 `makePrintable` 的字符串化逻辑均被 `__DEV__` 守卫，生产构建时会被 tree-shake 掉，不产生额外体积。
- 去重机制：`storedLogs: Dictionary<boolean>` 以消息字符串为 key 缓存是否已输出过，配合 `onlyOnce` 参数实现“同一消息仅输出一次”的行为。
- 安全打印：`makePrintable` 对 `undefined/NaN/Infinity/Date/RegExp/Function` 做模糊序列化，复杂对象尝试 `JSON.stringify` 并递归处理循环引用风险，异常时回退为 `?`，避免抛出错误阻断渲染。

此外，`src/util/styleCompat.ts` 中的 `warnDeprecated(deprecated, insteadApproach)` 是另一个独立的废弃 API 告警入口，使用本地 `deprecatedLogs` 缓存键 `deprecated + '^_^' + insteadApproach` 进行去重，直接调用 `console.warn` 输出 `[ECharts] DEPRECATED: ...`。

## 2. 关键文件与包

| 文件 | 作用 |
|---|---|
| `src/util/log.ts` | 统一的 ECharts 日志门面：`log`/`warn`/`error`/`deprecateLog`/`deprecateReplaceLog`/`makePrintable`/`throwError` |
| `src/util/styleCompat.ts` | 废弃样式 API 的专用告警函数 `warnDeprecated` |
| `src/animation/customGraphicKeyframeAnimation.ts` | 通过 `import { warn } from '../util/log'` 使用统一日志 |
| `src/animation/customGraphicTransition.ts` | 通过 `import { warn } from '../util/log'` 校验 style 属性合法性并告警 |
| `src/animation/universalTransition.ts` | 大数据量下禁用通用过渡时发出 `warn` |
| `src/chart/bar/BarView.ts` | 坐标系不支持或 `realtimeSort` 条件不满足时发出 `warn` |
| `src/chart/boxplot/boxplotTransform.ts` | 使用 `throwError`、`makePrintable` 构造错误上下文信息 |
| 其他组件中的 `console.warn/console.error` | 部分旧代码仍直接调用原生 console，尚未迁移到 `util/log` |

## 3. 架构与约定

- **单一入口**：新增业务日志应优先导入 `src/util/log` 中的 `log`/`warn`/`error`，而非直接使用 `console.*`。这保证了统一的前缀、去重和环境裁剪。
- **级别划分**：`log` 用于一般提示，`warn` 用于配置/使用不当的警告，`error` 用于不可恢复的错误；`deprecateLog`/`deprecateReplaceLog` 专用于废弃 API 的迁移提示，且默认带 `onlyOnce` 去重。
- **开发期增强**：`makePrintable(...hintInfo)` 仅在 `__DEV__` 下拼接可读字符串，供 `throwError` 或调试场景使用，将任意参数序列化为空格分隔的提示文本。
- **全局前缀**：所有经 `outputLog` 输出的消息都会前置 `[ECharts] `，方便在控制台快速筛选。
- **渐进迁移**：仓库中仍存在多处直接 `console.warn/console.error` 的散点调用（如 `chart/funnel/funnelLayout.ts`、`coord/geo/geoCreator.ts`、`component/radar/RadarView.ts` 等），说明日志体系仍在逐步收敛至 `util/log`。

## 4. 约定与约束

- **必须通过 `__DEV__` 包裹的废弃告警路径才会在生产构建中出现**：`deprecateLog`、`deprecateReplaceLog`、`warnDeprecated`、`makePrintable` 内部均有 `if (__DEV__)` 守卫，确保生产包不包含这些分支。
- **重复消息抑制**：`util/log` 的 `warn/log/error` 支持传入 `onlyOnce` 标志（例如 `warn('End frame with percent: 1 is missing in the keyframeAnimation.', true)`），利用 `storedLogs` 保证相同字符串只输出一次；`styleCompat.warnDeprecated` 则用组合键去重。
- **禁止在生产环境抛错**：`throwError` 虽然存在，但通常只在 `__DEV__` 路径中被 `makePrintable` 辅助构造错误信息时使用，生产构建中相关代码会被移除。
- **消息格式约定**：所有经 `outputLog` 输出的消息均以 `[ECharts] ` 开头，便于用户过滤；废弃 API 告警额外带上 `DEPRECATED:` 标记。
- **未强制覆盖全部 console 调用**：仓库并未通过 ESLint 规则禁止直接使用 `console.*`，因此当前约束更多是工程约定而非硬性检查——新代码应遵循统一入口，已有散点调用属于历史遗留。

总体而言，ECharts 的日志系统是一个轻量级、无依赖、围绕浏览器 `console` 封装的实用层，核心目标是：统一前缀、开发期友好提示、废弃 API 可追踪、生产构建零开销。