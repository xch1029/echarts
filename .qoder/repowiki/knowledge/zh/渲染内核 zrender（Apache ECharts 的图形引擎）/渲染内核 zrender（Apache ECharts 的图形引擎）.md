---
kind: external_dependency
name: 渲染内核 zrender（Apache ECharts 的图形引擎）
slug: zrender
category: external_dependency
category_hints:
    - vendor_identity
    - client_constraint
scope:
    - '**'
---

### 身份与角色
- zrender 是 Apache ECharts 的底层图形渲染引擎，提供 Canvas/SVG 图元、事件系统、动画等能力；ECharts 本身不直接操作 DOM/Canvas，而是通过 zrender 的 Path、Graphic、Handler 等抽象完成绘制。

### 集成方式
- 运行时：echarts 通过 `lib/` 产物在浏览器中消费 zrender 的运行时 API。
- 开发时：`build:lib` 用根 `tsconfig.json` 同时编译 echarts 和 zrender 源码，故 zrender 的类型声明必须随 echarts 一起可用（见 q4/q5 排查过程）。

### 稳定约束
- Windows 下构建脚本对 `fs.renameSync` 做 EPERM/EACCES/EBUSY 退避重试（`build/pre-publish.js`），因为 zrender 转换后一次性写入大量文件会触发杀毒软件句柄争用。
- pnpm 严格模式下需要 `preserveSymlinks: true` 才能让 TS 生成可移植的 `zrender/src/...` 模块说明符，避免 TS2742 "not portable" 错误。
- 验证具体 API/参数以 zrender 官方文档为准。