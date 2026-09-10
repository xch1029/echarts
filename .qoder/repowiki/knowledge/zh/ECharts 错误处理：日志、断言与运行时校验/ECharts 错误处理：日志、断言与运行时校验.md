---
kind: error_handling
name: ECharts 错误处理：日志、断言与运行时校验
category: error_handling
scope:
    - '**'
source_files:
    - src/util/log.ts
    - src/core/echarts.ts
    - src/model/Global.ts
    - src/chart/heatmap/HeatmapView.ts
    - src/chart/line/LineSeries.ts
    - src/chart/lines/LinesSeries.ts
    - extension-src/bmap/BMapCoordSys.ts
---

## 1. 整体方案

ECharts 没有引入统一的异常类型体系或中间件式错误处理框架，而是采用**“开发期断言 + 运行期 warn/error 日志 + 关键路径 throw”**的三层策略：

- **开发期断言**：通过 `zrender/src/core/util` 暴露的 `assert` 在 `__DEV__` 环境下做参数/状态校验，生产环境直接剔除。
- **运行期告警**：统一通过 `src/util/log.ts` 的 `warn` / `error` / `log` 输出带 `[ECharts]` 前缀的提示，支持 `onlyOnce` 去重。
- **致命错误**：对配置不合法、组件缺失、坐标系不支持等不可恢复场景，直接在调用点 `throw new Error(...)` 或调用 `util/log.ts` 的 `throwError`。

该模式贯穿核心入口（`setOption`/`dispatchAction`）、模型合并（`GlobalModel`）、各图表 View/Series 以及扩展模块（如 `extension-src/bmap`）。

## 2. 关键文件与职责

| 文件 | 职责 |
|---|---|
| `src/util/log.ts` | 唯一日志门面：`log`/`warn`/`error`/`deprecateLog`/`deprecateReplaceLog`/`makePrintable`/`throwError`。所有 console 输出统一加 `[ECharts]` 前缀，`onlyOnce` 基于内存 `storedLogs` 去重，`deprecate*` 仅在 `__DEV__` 生效。 |
| `src/core/echarts.ts` | ECharts 实例主循环；在 `setOption`/`setTheme`/`_onframe` 中用 try/catch 包裹生命周期，保证 `IN_EC_CYCLE_KEY` 标志位正确复位后重新抛出异常；同时检查 `disposed` 并调用 `disposedWarning`。 |
| `src/model/Global.ts` | 选项合并与组件注册中心；缺失组件/系列时通过 `error` 打印导入指引，且使用 `componetsMissingLogPrinted` 去重；`assertSeriesInitialized` 在缺少 series 时抛错。 |
| `src/chart/*`、`src/component/*`、`src/coord/*` | 具体图表/组件在自身校验失败时直接 `throw new Error`（如 Heatmap 必须配合 visualMap、Line 仅支持 cartesian/polar、Lines data 格式非法等）。 |
| `extension-src/bmap/BMapCoordSys.ts` | 扩展模块同样遵循：BMap API 未加载或重复注册时 `throw new Error`。 |

## 3. 架构与约定

### 3.1 日志与告警
- 所有用户可见的错误/警告都走 `src/util/log.ts`，禁止直接 `console.error/warn`（部分旧代码仍直接 `console.warn`，属于遗留用法）。
- `warn`/`error` 支持第二个参数 `onlyOnce: boolean`，同一消息只输出一次，避免大数据量渲染刷屏。
- `deprecateLog` / `deprecateReplaceLog(oldOpt, newOpt, scope?)` 用于废弃 API 提示，受 `__DEV__` 守卫。
- `makePrintable(...hintInfo)` 在 dev 下安全地序列化对象（处理 `NaN`/`Infinity`/`Date`/函数/正则），供调试信息构造。

### 3.2 断言（assert）
- 依赖 zrender 的 `assert(condition, message)`，仅在 `__DEV__` 生效，用于快速失败（如 `option is null/undefined`、`please use chart.getOption()`、`Option should contains series`）。
- 多处使用 `assert` 作为内部契约校验（如 pipeline 上游输出范围一致、model 非空等）。

### 3.3 运行时错误抛出
- **配置类错误**：如 `Heatmap must use with visualMap`、`Heatmap on cartesian must have two category axes`、`Line not support coordinateSystem besides cartesian and polar`、`Invalid data format` 等，直接 `throw new Error`。
- **组件缺失**：不在 `throw` 分支，而是通过 `error` 打印 `Component xxx is used but not imported...` 并跳过后续处理，保证库可继续运行。
- **扩展依赖缺失**：`BMap api is not loaded`、`Only one bmap component can exist` 等。

### 3.4 生命周期中的异常传播
`src/core/echarts.ts` 的核心更新流程（`prepareAndUpdate` → `updateMethods.update` → render）被 try/catch 包裹，确保：
- 无论是否抛出异常，`IN_EC_CYCLE_KEY` 都会复位为 `false`。
- `PENDING_UPDATE` 会被清空。
- 异常会向上传播给调用方（`setOption`/`dispatchAction`/`resize` 等）。
- 事件触发（`flushPendingActions`、`triggerUpdatedEvent`）只在正常路径执行，避免在异常状态下派发事件。

### 3.5 已处置实例保护
`ECharts` 实例在被 `dispose` 后，再次调用事件监听等方法会调用 `disposedWarning(id)` 并静默返回，而不是抛错。

## 4. 约定与约束

- **禁止在生产环境输出调试信息**：`deprecateLog`、`makePrintable` 等逻辑均受 `__DEV__` 守卫，构建时会移除。
- **告警去重**：组件缺失、重复 keyframe 等高频场景使用 `onlyOnce` 或 `componetsMissingLogPrinted` 记录，避免控制台风暴。
- **致命错误 vs 可恢复错误**：配置/数据非法 → `throw new Error`；组件未 import → `error` 提示并跳过；API 误用（如在 EC_CYCLE 内调用 `setOption`）→ `error` 提示并 return。
- **无自定义 Error 子类**：仓库未定义 `EChartsError` 等专用异常类型，全部使用原生 `Error` 字符串消息。
- **无全局 catch/recover 机制**：异常由调用方（用户代码）捕获，ECharts 内部只做必要的资源清理（重置 cycle 标志、清空 pending update）。
- **测试侧覆盖**：`test/ut` 与大量 HTML demo 覆盖了各类边界情况（如 `line-crash.html`、`axis-break*.html`、`candlestick-empty.html` 等），但未见专门的 error-handling 单元测试套件。

## 5. 适用性说明

本仓库是前端可视化库，不存在后端中间件式的错误处理管道，因此“middleware error handling”不适用；但其日志、断言、运行时校验与生命周期异常传播构成了完整的错误处理体系，适用于本分类。