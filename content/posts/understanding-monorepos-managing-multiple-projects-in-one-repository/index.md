---
title: "理解 Monorepo：在单个仓库中管理多个项目"
date: 2026-05-06T19:37:07+08:00
draft: false
tags: ["programming", "monorepo"]
comments: true
---
<center>
    <img src="./cover.png" alt="Monorepos" title="Monorepos" style="width: 100%;">
</center>

### 引言：Monorepo 是什么？
与将各项目分别存放在独立仓库的多仓库（Multi-repo / Polyrepo）模式不同，Monorepo（单体仓库）是一种将多个项目及其共享依赖统一集中在单个代码仓库中进行管理的架构模式，通过统一的目录结构与工具链管理，实现代码的高效组织与协作。单体仓库中的项目可以是相互连接的库、服务、应用程序，甚至是文档。
> A monorepo is a single repository containing multiple distinct projects, **with well-defined relationships**. —— [monorepo.tools](https://monorepo.tools/)

典型的 Monorepo 目录结构：
```
repo/
    apps/
        web/        <- Next.js 应用
        mobile/     <- React Native 应用
    packages/
        ui/         <- 共享组件库
        utils/      <- 工具函数
        config/     <- 共享配置
    package.json
```

Monorepo 特点
- 单一 Git 仓库
- 多个独立项目/包
- 共享工具链与依赖
- 跨包原子化提交

Polyrepo 特点
- 每个项目独立仓库
- 各自管理依赖版本
- 跨项目变更需多次 PR
- 工具配置难以统一

Monorepo 也不同于 Monolith（单体应用），后者是将多个子模块直接合并为一个整体应用系统；而在单体仓库中，每个子项目可以是独立的应用程序，也可以是可复用的包（如函数库、组件、服务），项目之间存在明确的关联。
### 为什么要使用 Monorepo？
1. 当团队规模扩大、项目数量增加且跨项目代码复用需求提升时，多仓库模式的局限逐渐显现，例如版本对齐困难、工具链配置重复以及跨仓库协作成本高等问题。
2. AI 编码代理重塑软件开发的背景下，这些问题被进一步放大。在多仓库模式下，跨仓库开发往往伴随上下文丢失、配置重复以及跨领域变更需要手动协调等问题；而 AI 编码代理对全局上下文的依赖。Monorepo 通过统一代码空间与依赖结构，将相关上下文集中管理，从而降低跨模块协作与自动化修改的复杂度。

Monorepo 因此被广泛采用，其核心优势体现在：
- 依赖统一管理：所有项目依赖集中维护，避免不同项目因使用不同版本导致兼容问题。
- 原子化提交：跨多个包/项目的关联修改可在一次 commit 中完成，保证 Git 历史的完整性与可追溯性。
- 代码共享与复用：不同项目之间可以低成本共享组件、工具函数或公共模块，减少重复开发。
- 高效跨项目重构：支持在同一仓库内进行全局范围的结构调整，降低大规模重构的复杂度与风险。
- 统一工具链：如 ESLint, Prettier, Vitest 以及 CI 配置等可集中维护，一次配置即可在所有项目中生效，显著减少重复配置成本。

从 AI Agent 的视角来看，Monorepo 优势更加明显。一方面，AI Agent需要在更大范围内理解代码上下文，Monorepo 将多个项目与共享模块集中在同一仓库中，使其能够获取更完整的依赖关系与调用链，从而显著提升代码生成、重构与问题定位的准确性；另一方面，Monorepo 所强调的统一依赖管理与标准化工具链，也为 AI 自动化执行任务（如批量重构、跨模块测试修复、依赖升级等）提供了更稳定、一致的执行环境，降低了跨仓库协作的复杂度与不确定性。

不过，Monorepo 并不是灵丹妙药，在带来便利的同时，也引入了新的复杂性。如构建与 CI 耗时增加、版本管理与发布复杂度上升、IDE/Git操作在超大仓库下响应变慢、难以实现细粒度访问隔离、依赖升级的连锁风险等。对于团队极小、项目间几乎不共享代码的场景，Polyrepo 模式仍然简单高效。

### Monorepo 中的 Workspaces
Workspaces（工作区）是包管理器原生支持的 Monorepo 基础机制。它让包管理器理解仓库中存在多个子包，并将它们之间的本地引用自动 symlink，使 `import '@myorg/ui'` 直接指向本地源码，而非远程 npm 包。
- npm workspaces

    npm v7+ 支持 workspaces，配置写在根目录的 `package.json` 中，无需额外文件。
    ```json
    {
        "name": "my-monorepo",
        "private": true,
        "workspaces": [
            "apps/*",
            "packages/*"
        ]
    }
    ```
    声明后，运行 `npm install` 时，npm 会自动将 `packages/ui` 等本地包 symlink 到根目录的 `node_modules/`，无需手动 link。

    常用命令：

    ```shell
    # 在指定 workspace 运行脚本
    npm run build --workspace=packages/ui

    # 为某个 workspace 安装依赖
    npm install react --workspace=apps/web

    # 运行所有 workspace 的同名脚本
    npm run test --workspaces
    ```

- pnpm workspaces

    pnpm 通过独立的 `pnpm-workspace.yaml` 文件声明工作区，同时以硬链接 + 符号链接的方式管理 `node_modules`，彻底解决幽灵依赖问题。
    ```yaml
    # pnpm-workspace.yaml
    packages:
        - "apps/*"
        - "packages/*"
    ```
    pnpm 的 `node_modules` 结构与 npm/yarn 不同：每个包只能访问自己 `package.json` 中声明的依赖，未声明的包无法被意外引用（即消除幽灵依赖）。

    常用指令：
    ```shell
    # 在指定 workspace 运行脚本
    pnpm --filter packages/ui build

    # 为某个 workspace 安装依赖
    pnpm add react --filter apps/web

    # 递归运行所有 workspace 的脚本
    pnpm -r run test

    # 安装本地包作为依赖（使用 workspace 协议）
    pnpm add @myorg/ui --filter apps/web --workspace
    ```
    在引用本地包时，推荐使用 workspace 协议，明确表达这是本地依赖而非远程包：
    ```json
    // apps/web/package.json
    {
        "dependencies": {
            "@myorg/ui": "workspace:*"
        }
    }
    ```

### 科技巨头的 Monorepo 实践
Monorepo 被科技巨头广泛采用，用于提升协作效率、代码复用与工程规模管理能力。

Google 通过超大规模单体代码库 [Piper](https://en.wikipedia.org/wiki/Piper_(source_control_system)) 推动了这一模式的普及，并强调“共享所有权”的开发文化，使工程师可以跨项目协作。同时，Google 研发了构建工具 [Bazel](https://opensource.google/projects/bazel)，通过增量构建机制显著提升大型代码库的构建效率。
> [mega](https://github.com/web3infra-foundation/mega) 是一个面向 AI 时代的开源 Piper 实现。

Facebook 同样采用 Monorepo 管理其核心产品（如 Facebook,Instagram, WhatsApp 等）。其“move fast and break things”的工程文化配合 [Buck](https://github.com/facebook/buck) 构建系统，使大规模代码构建保持高效与一致性。

Microsoft 则将 Windows 代码库迁移至 Git 单体仓库，并通过 [VFS for Git](https://github.com/microsoft/vfsforgit) 解决超大规模仓库的性能问题，实现了在巨型代码库中的高效开发与协作。