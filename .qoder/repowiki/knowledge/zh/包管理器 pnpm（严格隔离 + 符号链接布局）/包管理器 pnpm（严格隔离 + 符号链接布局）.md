---
kind: external_dependency
name: 包管理器 pnpm（严格隔离 + 符号链接布局）
slug: pnpm
category: external_dependency
category_hints:
    - framework_behavior
    - client_constraint
scope:
    - '**'
---

### 身份与角色
- 本仓库使用 pnpm 作为包管理器，其默认严格隔离布局与上游 npm（package-lock.json）不同，导致两类常见差异：
  2) `@types/*` 全局类型只包含根目录显式声明的包，`@types/node` 作为传递依赖在 pnpm 下不可见，需在 devDependencies 显式声明以复刻 npm 提升行为。

### 稳定约束
- `prepare` 钩子执行 `npm run build:lib && husky install`，安装后自动触发构建。
- 当启用 `preserveSymlinks: true` 时，TS 解析 `@types/jest` 内部 import 只能查到根 `node_modules`，需把 `jest-diff`、`pretty-format` 也显式声明为直接依赖来对齐 npm 布局。
- 这些约束影响所有基于该仓库的开发环境搭建，新增 build 工具链依赖时必须显式声明到 devDependencies。