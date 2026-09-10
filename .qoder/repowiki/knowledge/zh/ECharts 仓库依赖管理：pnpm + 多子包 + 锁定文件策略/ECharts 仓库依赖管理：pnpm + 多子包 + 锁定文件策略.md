---
kind: dependency_management
name: ECharts 仓库依赖管理：pnpm + 多子包 + 锁定文件策略
category: dependency_management
scope:
    - '**'
source_files:
    - package.json
    - pnpm-lock.yaml
    - i18n/package.json
    - theme/package.json
    - test/runTest/package.json
    - extension-src/bmap/tsconfig.json
---

## 1. 使用的系统/工具

- **包管理器**：使用 **pnpm**（由根目录 `pnpm-lock.yaml` 的 lockfileVersion 9.0 确认），所有依赖声明集中在根 `package.json`。
- **锁定文件**：提交并受版本控制的 `pnpm-lock.yaml`，用于保证全仓依赖树可重现安装。
- **构建与发布**：通过自定义 Node 脚本 `build/build.js`、`build/build-lib.js`、`build/build-i18n.js` 等完成源码编译、多入口产物打包；发布产物为 npm 包（`name: "echarts"`）。
- **工作区结构**：仓库采用“单根 package.json + 多个独立子包”的多包模式，而非 pnpm workspace monorepo。每个子包拥有自己的 `package.json`，但仅做最小化元数据声明。

## 2. 关键文件与位置

| 文件 | 作用 |
|---|---|
| `package.json` | 根包声明，定义运行时依赖、开发依赖、`scripts`、`exports` 多入口映射、`sideEffects` 等 |
| `pnpm-lock.yaml` | pnpm 依赖锁定文件，记录精确版本与完整依赖图 |
| `i18n/package.json` | 国际化资源子包，仅声明 `type: commonjs` |
| `theme/package.json` | 主题资源子包，仅声明 `type: commonjs` |
| `test/runTest/package.json` | 视觉回归测试子包，单独声明 puppeteer、pixelmatch 等测试依赖 |
| `extension-src/bmap/tsconfig.json` | 百度地图扩展的 TypeScript 配置（该扩展无独立 package.json，随主包构建） |
| `ssr/client/index.js` / `index.d.ts` | SSR 客户端子模块，作为 npm `exports` 中 `./ssr/client/index` 暴露 |

## 3. 架构与约定

### 3.1 依赖分层
- **运行时依赖（dependencies）**：仅两个——`tslib`（TS 运行时辅助）和 `zrender`（底层渲染引擎）。这表明 ECharts 对外暴露的 API 不直接依赖其他第三方库，所有图形绘制能力通过 `zrender` 抽象。
- **开发依赖（devDependencies）**：涵盖 TypeScript、Rollup、Babel、ESLint、Jest、Terser、Husky、API Extractor 等构建/测试/类型工具链。
- **子包零依赖**：`i18n` 与 `theme` 子包的 `package.json` 只声明 `type: commonjs`，不包含任何依赖，表明它们是纯资源/JS 模块，不参与依赖解析。

### 3.2 多入口与按需加载
根 `package.json` 的 `exports` 字段显式声明了数十个路径映射（`./core`、`./charts`、`./components`、`./features`、`./renderers`、`./index.blank`、`./index.common`、`./index.simple`、`./lib/*`、`./extension/dataTool/*`、`./extension/bmap/*`、`./ssr/client/index` 等），配合 `sideEffects` 数组，使消费者可按需引入特定图表/组件/渲染器，从而控制最终产物体积。

### 3.3 子包与主包的关系
- `i18n` 与 `theme` 是独立的 npm 子包，但它们的 `package.json` 没有 `name`/`version`/`main`，实际是通过构建流程将内容复制到发布产物目录，再由根包的 `exports` 暴露给消费者。
- `extension-src/bmap` 与 `extension-src/dataTool` 是 TypeScript 源码，经 `build/build.js --type extension` 构建后输出到 `extension/` 目录，并通过 `./extension/*` 暴露。
- `ssr/client` 有独立源码与类型声明，通过 `./ssr/client/index` 暴露，并在 `exports` 中分别指定 `import` 与 `require` 入口。

### 3.4 测试子包隔离
`test/runTest/` 下有自己的 `package.json`，包含 `puppeteer`、`pixelmatch`、`pngjs`、`serve-handler` 等视觉回归测试专用依赖，与主包开发依赖解耦，避免污染主包依赖树。

## 4. 约定与约束

- **单一来源的依赖声明**：所有依赖统一在根 `package.json` 的 `dependencies` 与 `devDependencies` 中声明，子包不重复声明业务依赖，保持依赖集中管理。
- **锁定文件强制一致**：`pnpm-lock.yaml` 已提交至仓库，安装时应基于该锁定文件生成 `node_modules`，确保不同环境依赖树一致。
- **运行时依赖极简**：对外发布的 ECharts 包仅依赖 `tslib` 与 `zrender`，第三方依赖被严格收敛，降低下游消费者的依赖冲突风险。
- **子包仅声明模块类型**：`i18n` 与 `theme` 子包通过 `package.json` 中的 `type: commonjs` 明确其 CommonJS 模块格式，避免 ESM/CJS 混用问题。
- **按需引入通过 `exports` 约束**：所有可被外部 import 的路径均在 `exports` 中显式列出，未列出的路径不会被当作公开 API 暴露，形成隐式的“白名单”约束。
- **husky 钩子参与依赖生命周期**：`prepare` 脚本执行 `npm run build:lib && husky install`，在安装阶段自动构建 lib 并安装 Git hooks，确保本地开发与 CI 行为一致。
- **无 vendoring**：未发现 `vendor/` 或内联第三方源码的做法，所有第三方代码均通过 pnpm 从 npm 注册表拉取。
- **无私有 registry/GPR 配置**：仓库中未发现 `.npmrc`、`.pnpmrc`、`GOPRIVATE` 等私有源配置，默认使用公共 npm 源。
