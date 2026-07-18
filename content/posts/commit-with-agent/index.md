---
title: "智能体时代下的代码提交"
date: 2026-07-18T18:54:25+08:00
draft: false
tags: ["ai"]
comments: true
---
最近马斯克的编码智能体 `grok-build` 开源了，看了一下，发现初始[提交](https://github.com/xai-org/grok-build/commit/c68e39f60462f28d9be5e683d9cbe2c57b1a5027)直接是产品的一个版本，无法查看仓库的提交记录和设计考量。毫无疑问，这会是未来智能体编码时代的一个典型特征：代码的大部分工作是由智能体完成的，而智能体单次对代码的更改可能远超常人可接受的范围（即使可以接受，越来越多的人也不愿花费太多时间用于审核 AI 生成的代码），因此，Vibe Coding Project 的初始版本大概率是一个较为成熟的版本。

这种模式存在一个特点，那就是无论是人还是智能体对仓库的理解高度依赖于项目文档，如果仓库代码量非常大，那简洁的文档和层次化的仓库结构无疑会降低人类认知负担与智能体认知成本。另外一个特点是什么呢？我想可能是回退困难。设想这样一个场景，我们发现了一个潜在的问题（A时刻），通过指导智能体完成了问题修复（B时刻），后续又经历了在此基础上的多轮功能开发和问题修复（C时刻），倘若过程中经历了上下文压缩，那么在没有提交历史的情况下，想要回退到A时刻无疑非常困难。因此，我认为清晰而又明确的提交历史对智能体而言同样是不可缺少的，但目前来看，很多有智能体在编码时主动做提交，不确定这方面是否需要在训练阶段下一些功夫。另一种方式是人类触发的代码提交，即人类通过对话直接告诉智能体“是时候做提交了”。

加上最近学习了 `git add -p` 的用法，我设计了一个用于提交的技能：[commit-skill](https://github.com/Youpen-y/commit-skill)。

````markdown
---
name: commit
description: Commits changes in small, focused batches — one logical change per commit. Use when the user wants to commit changes.
---

# Conventional Commit

## Message Format

```
<type>(<scope>): <description>
```

- **type** (required): `feat` | `fix` | `docs` | `style` | `refactor` | `perf` | `test` | `build` | `ci` | `chore` | `revert`
- **scope** (optional): lowercase alphanumeric + hyphens
- **breaking**: append `!` before `:`, e.g. `feat(api)!: remove old endpoint`
- **description**: lowercase start, no trailing period, imperative mood, use plain common words, keep short
- **body** (optional): blank line separated, explain *what* and *why*
- **footer** (optional): `Key: Value` lines, e.g. `BREAKING CHANGE: ...`, `Closes #12`

## Splitting Principles

One commit = one logical change. When the working tree mixes multiple concerns:

- **Split across files** — stage only the files for one concern.
- **Split within a file** — when a single file mixes concerns, stage individual hunks (Step 2b).
- **Every intermediate commit must build and pass tests.** Never commit a broken state. If a split breaks the build mid-way, fix the boundary or merge the units.
- **Respect dependency order.** If unit B uses code introduced by unit A, commit A first.
- **Don't over-split.** Keep units together when hunks interleave concerns on the same lines, when splitting would need artificial stubs to compile, or when the units are inseparable (e.g., a new API plus its only caller).

## Workflow

Repeat the following steps until all changes are committed:

### 1. Check current changes

```bash
git status --short
git diff --stat
git diff --cached --stat
```

Review what has changed. If no changes remain, stop.

### 2. Identify logical units and dependency order

Read `git diff` (unstaged) and `git diff --cached` (staged). Group changes into coherent units — one per commit — and order them so each builds on what came before (dependencies first). If there is only one unit, go to Step 2b.

### 2a. Stage the unit — file level

If the unit maps cleanly to whole files:

```bash
git add <file1> <file2> ...
```

Do **not** `git add .` or `git add -A`.

### 2b. Stage the unit — within a file (patch staging)

When a file mixes concerns, stage individual hunks. A human would use `git add -p <file>` interactively (`y/n/s/e`); as an agent, build a patch and apply it to the index:

```bash
# 1. Inspect remaining unstaged hunks for the file
git diff -- <file>

# 2. Write a valid unified diff with only THIS unit's hunks
#    (keep the `diff --git` header, --- / +++ lines, and chosen @@ hunks
#     with their context lines) -> /tmp/unit.patch

# 3. Apply those hunks to the index
git apply --cached --recount /tmp/unit.patch

# 4. Confirm what's staged
git diff --cached -- <file>
```

Repeat until all hunks for this unit are staged. Only hunks belonging to this one logical change go in.

### 2c. Verify the intermediate state builds and tests pass

Before committing — especially after a within-file split — confirm the would-be commit is green:

```bash
# Hide unstaged changes; working tree now equals the would-be commit
git stash push --keep-index --include-untracked

# Run build and tests
#   <build> && <test>

# Restore unstaged changes
git stash pop
```

If it fails, the boundary is wrong. Fix it by adding hunks this commit needs (e.g., a function definition it calls), reordering so dependencies land first, or merging the unit with its dependency into one commit. Proceed only once the state is green — or the unit is trivially safe (e.g., docs-only).

### 3. Compose and validate the message

Analyze the staged diff (`git diff --cached`) and compose a commit message. Validate it with `validate.sh`, which lives next to this `SKILL.md`. Run it by the absolute path resolved against the **skill directory**:

```bash
/path/to/skill-dir/validate.sh "<message>"
```

If validation fails, fix the message and re-validate.

### 4. Commit

```bash
# Single-line message
git commit -m "<message>"

# Multi-line message (body / footer)
git commit -F <(printf '%s\n' "<message>")
```

### 5. Confirm and loop

```bash
git log -1 --oneline
```

Then go back to **step 1** to handle remaining changes.

````

基本的逻辑是一个循环，通过在循环中不断识别逻辑单元进行拆分，提交前验证构建状态和测试通过，实现大量代码的精细化提交，以保留详细的提交历史。

