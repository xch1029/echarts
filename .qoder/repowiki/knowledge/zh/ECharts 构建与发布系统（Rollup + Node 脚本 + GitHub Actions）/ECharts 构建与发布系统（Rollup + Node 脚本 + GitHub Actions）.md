---
kind: build_system
name: ECharts 构建与发布系统（Rollup + Node 脚本 + GitHub Actions）
category: build_system
scope:
    - '**'
source_files:
    - package.json
    - tsconfig.json
    - .github/workflows/ci.yml
    - .github/workflows/source-release.yml
    - .github/workflows/nightly.yml
    - build/build.js
    - build/build-i18n.js
    - build/build-lib.js
    - build/dev-fast.js
    - build/testDts.js
    - build/checkHeader.js
    - build/source-release/prepareReleaseMaterials.js
    - build/nightly/prepare.js
    - build/nightly/post.js
    - test/build/mktest.js
---

## 1. 使用的构建系统与工具

- **包管理器**：pnpm（`pnpm-lock.yaml`），npm scripts 通过 `package.json` 暴露所有构建命令。
- **打包器**：Rollup（`rollup: 2.34.2`）配合 `@rollup/plugin-commonjs`、`@rollup/plugin-node-resolve`、`@rollup/plugin-replace`、`@rollup/plugin-terser`，用于将 TypeScript 源码编译并打包为多种产物。
- **压缩器**：Terser（`terser: ^5.16.1`），在 `--min` 模式下对输出进行压缩。
- **类型声明生成**：TypeScript（`typescript: 4.4.3`，`target: ES3`）+ `api-extractor`（`@microsoft/api-extractor: 7.31.2`）+ `@lang/rollup-plugin-dts`，分别产出 `.d.ts` 和 `.d.cts`。
- **开发辅助**：esbuild（`esbuild: ^0.8.39`）、magic-string、chalk、commander；本地开发使用 `concurrently` + `http-server` 启动 dev server。
- **测试框架**：Jest（`jest: ^26.6.1` + `ts-jest`）运行单元测试；Puppeteer 驱动的视觉回归测试位于 `test/runTest/`。
- **CI/CD**：GitHub Actions（`.github/workflows/`），Node.js 20.x 环境。

## 2. 关键文件与脚本

- `package.json`：定义全部 npm scripts（`build`、`build:esm`、`build:i18n`、`build:lib`、`build:extension`、`build:ssr`、`dev`、`release`、`test`、`test:dts`、`lint`、`checktype` 等）以及 `exports` 字段中声明的多入口（`./core`、`./charts`、`./components`、`./features`、`./renderers`、`./index.blank/.common/.simple`、`./ssr/client/index`、`./extension/*`、`./theme/*`、`./i18n/*`）。
- `tsconfig.json`：统一 TS 编译配置（`target: ES3`、`outDir: lib`、`declaration: true`、`preserveSymlinks: true`），包含 `src/**`、`extension-src/**`、`ssr/client/**`、`test/lib/**`。
- `build/build.js`、`build/build-i18n.js`、`build/build-lib.js`、`build/dev-fast.js`、`build/testDts.js`、`build/checkHeader.js` 等 Node 脚本（由 `package.json` scripts 调用）。
- `build/source-release/prepareReleaseMaterials.js`：生成 Apache 源发布材料。
- `build/nightly/prepare.js`、`build/nightly/post.js`：Nightly 预发布流程。
- `test/build/mktest.js`：根据示例 HTML 生成测试用例的构建脚本。
- CI 工作流：
  - `.github/workflows/ci.yml`：PR 触发 lint → 构建类型 → `checktype` → 单元测试 → `npm run release` → `test:dts`。
  - `.github/workflows/source-release.yml`：基于 `release` 事件准备并发布 Apache 源包，再下载后独立验证构建。
  - `.github/workflows/nightly.yml`：定时/手动触发 nightly 发布，替换 `zrender` 依赖为 `zrender-nightly` 后执行 `npm run release` 并发布到 npm。
  - `.github/workflows/nightly-next.yml`、`update-notice-year.yml`、`stale.yml` 等辅助流程。

## 3. 架构与约定

- **多入口产物**：通过 Rollup 按 `--type` 参数（`all`、`common`、`simple`、`extension`、`ssr`）和 `--format`（`cjs`、`esm`）组合生成不同体积与用途的 bundle，输出至 `dist/`（如 `echarts.js`、`echarts.min.js`、`echarts.common.js`、`echarts.esm.mjs`、`echarts.simple.*`、`extension/bmap.js`、`extension/dataTool.js`）。
- **ESM/CJS 双出口**：`package.json` 的 `exports` 同时声明 `import`（`index.js`）与 `require`（`dist/echarts.js`）路径，并提供子路径导出（`./core`、`./charts`、`./components`、`./features`、`./renderers`、`./index.blank/.common/.simple`、`./ssr/client/index`、`./extension/*`、`./theme/*`、`./i18n/*`）。
- **类型声明分离**：TS 编译输出到 `lib/`，再通过 api-extractor 生成 `types/dist/*.d.ts`、`types/dist/echarts.d.cts`，供消费者引用。
- **国际化资源构建**：`build/build-i18n.js` 将 `i18n/lang*.ts` 转换为可独立加载的 JS 模块，输出到 `i18n/`。
- **扩展与 SSR**：`extension-src/`（bmap、dataTool）与 `ssr/client/` 作为独立子项目参与构建，分别通过 `build:extension` 与 `build:ssr` 产出。
- **Side Effects 标记**：`package.json` 的 `sideEffects` 明确列出会被副作用加载的入口（`index.js`、`index.blank.js`、`index.common.js`、`index.simple.js`、`lib/echarts.js`、`lib/chart/*.js`、`lib/component/*.js`、`extension/**/*.js`、`theme/*.js`、`i18n/*.js`），便于 Tree-shaking。
- **版本管理**：版本号集中在 `package.json` 的 `version` 字段（当前 `6.1.0`），Nightly 流程通过 `build/nightly/prepare.js` 与 `post.js` 动态调整。

## 4. 约定与约束

- **构建命令约定**：所有构建均通过 `npm run <script>` 调用，禁止直接执行 `node build/*.js`（scripts 是官方入口）。
- **最小化开关**：生产构建需显式传入 `--min` 参数（见 `build`、`build:esm`、`build:extension` 脚本），否则仅生成未压缩产物。
- **增量 Lint**：CI 中通过 `git diff` 收集变更的 `*.ts` 文件，只对改动文件执行 ESLint，提升 PR 检查速度。
- **类型检查门禁**：CI 强制执行 `npm run checktype`（`tsc --noEmit` 针对 `src/`、`extension-src/bmap/tsconfig.json`），确保新增代码通过 TypeScript 严格模式。
- **DTS 验证**：CI 在 `npm run release` 之后执行 `npm run test:dts`，校验生成的类型声明可通过 dtslint。
- **Source Release 可重建性**：`source-release.yml` 将 `src/`、`extension-src/`、`ssr/client/src/`、`build/`（排除 `build/source-release`）、`test/ut`、`test/types`、`package.json`、`tsconfig.json` 等打包为源码制品，并在隔离目录中重新 `npm ci && npm run release && npm run test && npm run test:dts` 以验证可复现性。
- **Nightly 依赖替换**：Nightly 构建通过 `npm i zrender@npm:zrender-nightly` 临时替换核心渲染引擎依赖，确保 nightly 包与 zrender-nightly 同步发布。
- **Husky 钩子**：`prepare` script 自动执行 `husky install`，本地提交前受 `.husky/pre-commit` 约束。