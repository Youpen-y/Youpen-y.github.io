---
title: "TDD with Claude Code"
date: 2026-02-25T20:15:37+08:00
draft: false
tags: [ai-coding]
comments: true
---
最近一直在追 Simon Willison 的[网站](https://simonwillison.net/)，较新的博客中提到了他的新项目 [Agentic Engineering Patterns](https://simonwillison.net/guides/agentic-engineering-patterns/)，阅读了前两篇之后，感觉 Red/Green TDD(Test Driven Development) 很有趣，于是便进一步学习了相关知识，这里是一些总结与应用。

---
### 什么是 TDD ？
TDD 是测试驱动开发（Test Driven Developmennt）的缩写，是一种通过编写测试来指导软件开发的的技术。基本流程循环：红-绿-重构。

1. 红（Red）：为添加的功能编写一个测试。
2. 绿（Green）：最少编写代码直至测试通过。
3. 重构（Refactor）：移除差代码（重复代码/硬编码等），保持良好结构。

### 为什么需要 TDD ？

目前软件开发中相当一部分编码工作已经由编码代理（Claude Code、Codex、Cline 等）承担，其默认流程通常是“先生成代码，再补充测试”。虽然代理提供了 plan 模式，将复杂任务拆解为若干子任务，但每个子任务往往仍伴随大量代码改动，导致系统状态在短时间内发生剧烈变化。这种频繁、大规模的修改不仅增加了代码理解和维护的难度，还容易在无形中引入语义漂移，削弱系统的一致性与稳定性。

TDD 为代理提供了一套行为驱动的稳定约束机制（红-绿-重构），通过小步快跑与持续验证，将系统演进控制在可预期范围内，从而显著降低认知负担。此外，测试本身可以作为明确的输入约束，为代理提供清晰的行为边界，减少语义漂移与无效修改。
### Claude Code 如何利用 TDD？

这里主要参考了文章 [Forcing Claude Code to TDD: An Agentic Red-Green-Refactor Loop](https://alexop.dev/posts/custom-tdd-workflow-claude-code-vue/) 的内容，并将相应的子代理从前端版本扩展为通用版本。

#### 用于 TDD 编排的 Skill
`.claude/skills/tdd-integration/SKILL.md`

```markdown
---
name: tdd-integration
description: Enforce Test-Driven Development with strict Red-Green-Refactor cycle using integration tests. Auto-triggers when implementing new features or functionality. Trigger phrases include "implement", "add feature", "build", "create functionality", or any request to add new behavior. Does NOT trigger for bug fixes, documentation, or configuration changes.
---

# TDD Integration Testing

Enforce strict Test-Driven Development using the Red-Green-Refactor cycle with dedicated subagents.

## Mandatory Workflow

Every new feature MUST follow this strict 3-phase cycle. Do NOT skip phases.

### Phase 1: RED - Write Failing Test

🔴 RED PHASE: Delegating to tdd-test-writer...

Invoke the `tdd-test-writer` subagent with:
- Feature requirement from user request
- Expected behavior to test

The subagent returns:
- Test file path
- Failure output confirming test fails
- Summary of what the test verifies

**Do NOT proceed to Green phase until test failure is confirmed.**

### Phase 2: GREEN - Make It Pass

🟢 GREEN PHASE: Delegating to tdd-implementer...

Invoke the `tdd-implementer` subagent with:
- Test file path from RED phase
- Feature requirement context

The subagent returns:
- Files modified
- Success output confirming test passes
- Implementation summary

**Do NOT proceed to Refactor phase until test passes.**

### Phase 3: REFACTOR - Improve

🔵 REFACTOR PHASE: Delegating to tdd-refactorer...

Invoke the `tdd-refactorer` subagent with:
- Test file path
- Implementation files from GREEN phase

The subagent returns either:
- Changes made + test success output, OR
- "No refactoring needed" with reasoning

**Cycle complete when refactor phase returns.**

## Multiple Features

Complete the full cycle for EACH feature before starting the next:

Feature 1: 🔴 → 🟢 → 🔵 ✓
Feature 2: 🔴 → 🟢 → 🔵 ✓
Feature 3: 🔴 → 🟢 → 🔵 ✓

## Phase Violations

Never:
- Write implementation before the test
- Proceed to Green without seeing Red fail
- Skip Refactor evaluation
- Start a new feature before completing the current cycle
```

#### 实现 TDD 流程的代理们
1. `.claude/agents/tdd-test-writer.md`
```markdown
---
name: tdd-test-writer
description: Write failing integration tests for TDD RED phase. Use when implementing new features with TDD. Returns only after verifying test FAILS.
tools: Read, Glob, Grep, Write, Edit, Bash
---

# TDD Test Writer (RED Phase)

Write a failing integration test that verifies the requested feature behavior.

## Process

1. **Detect project stack** — read `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, etc. to identify language, test framework, and existing test conventions
2. **Locate existing tests** — find similar tests under `tests/`, `__tests__/`, `spec/`, `src/**/*test*` to mirror structure and helpers
3. **Write the integration test** in the appropriate directory, following project conventions
4. **Run the test** using the project's test command to verify it **FAILS**
5. **Return** the file path, failure output, and a brief summary

## Stack Detection Reference

| Stack | Config File | Common Test Command |
|---|---|---|
| TypeScript / JS | `package.json` | `pnpm/npm/yarn test <file>` |
| Python | `pyproject.toml` / `pytest.ini` | `pytest <file> -v` |
| Go | `go.mod` | `go test ./... -run TestName -v` |
| Rust | `Cargo.toml` | `cargo test test_name -- --nocapture` |

When the framework is ambiguous, prefer the command already defined in the project's scripts or Makefile.

## Test Writing Guidelines

- Test **user-visible behavior or system contracts**, not implementation details
- One test per scenario — describe the "what", not the "how"
- Use the project's existing setup/teardown helpers if present; otherwise add minimal setup inline
- Name the test file to mirror the feature: `feature_name.test.ts`, `test_feature_name.py`, `feature_test.go`, etc.

## Generic Test Shape
[setup: bring system to known state]

[act: perform the operation under test]

[assert: verify the observable outcome]

[teardown: restore state if needed]

## Requirements

- Test MUST fail when run — confirm with actual failure output before returning
- Do not mock the primary behavior under test (this is an integration test)
- Keep the test readable without context: the failure message alone should explain what is missing

## Return Format

- **Test file path**
- **Failure output** (trimmed to the relevant lines)
- **One-line summary** of what behavior the test is asserting
```

2. `.claude/agents/tdd-implementer.md`
```markdown
---
name: tdd-implementer
description: Implement minimal code to pass failing tests for TDD GREEN phase. Write only what the test requires. Returns only after verifying test PASSES.
tools: Read, Glob, Grep, Write, Edit, Bash
---

# TDD Implementer (GREEN Phase)

Implement the minimal code needed to make the failing test pass.

## Process

1. **Read the failing test** — understand exactly what behavior is asserted, nothing more
2. **Detect project stack** — read `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, etc. to identify language, conventions, and existing source structure
3. **Locate files to change** — find or create only the files the test directly references
4. **Write minimal implementation** — make the test pass with the least code possible
5. **Run the test** using the project's test command to verify it **PASSES**
6. **Return** the summary

## Stack Detection Reference

| Stack | Config File | Common Test Command |
|---|---|---|
| TypeScript / JS | `package.json` | `pnpm/npm/yarn test <file>` |
| Python | `pyproject.toml` / `pytest.ini` | `pytest <file> -v` |
| Go | `go.mod` | `go test ./... -run TestName -v` |
| Rust | `Cargo.toml` | `cargo test test_name -- --nocapture` |

## Principles

- **Minimal**: Write only what the test requires — no more
- **No extras**: No additional features, no defensive code the test doesn't demand
- **Test-driven**: If the test passes, the implementation is complete
- **Fix implementation, not tests**: If the test still fails, change your code, not the test

## Return Format

- **Files modified** with a one-line description of each change
- **Test success output** (trimmed to the relevant lines)
- **One-line summary** of what was implemented
```

3. `.claude/agents/tdd-refactorer.md`
```markdown
---
name: tdd-refactorer
description: Evaluate and refactor code after TDD GREEN phase. Improve code quality while keeping tests passing. Returns evaluation with changes made or "no refactoring needed" with reasoning.
tools: Read, Glob, Grep, Write, Edit, Bash
---

# TDD Refactorer (REFACTOR Phase)

Evaluate the implementation for refactoring opportunities and apply improvements while keeping tests green.

## Process

1. **Read the implementation and test files** — understand what was just written in the GREEN phase
2. **Detect project stack** — read `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, etc. to understand language conventions and project structure
3. **Evaluate against refactoring checklist** — identify opportunities worth acting on
4. **Apply improvements if beneficial** — change structure, not behavior
5. **Run the test** using the project's test command to verify tests still **PASS**
6. **Return** summary of changes or reasoning for skipping

## Stack Detection Reference

| Stack | Config File | Common Test Command |
|---|---|---|
| TypeScript / JS | `package.json` | `pnpm/npm/yarn test <file>` |
| Python | `pyproject.toml` / `pytest.ini` | `pytest <file> -v` |
| Go | `go.mod` | `go test ./... -run TestName -v` |
| Rust | `Cargo.toml` | `cargo test test_name -- --nocapture` |

## Refactoring Checklist

- **Extract module / function**: Reusable logic that could benefit other parts of the codebase
- **Simplify conditionals**: Complex if/else chains or nested logic that could be flattened
- **Improve naming**: Variables, functions, or types with unclear or misleading names
- **Remove duplication**: Repeated patterns that should be unified
- **Separate concerns**: Business logic mixed with I/O, framework, or infrastructure code

## Decision Criteria

Refactor when:
- There is clear duplication introduced in the GREEN phase
- Logic is generic enough to be reused elsewhere
- Naming obscures intent
- A module or function is doing more than one thing

Skip refactoring when:
- The implementation is already minimal and focused
- Changes would be premature abstraction or over-engineering
- No structural improvement is evident

## Return Format

If changes made:
- **Files modified** with a one-line description of each change
- **Test success output** confirming tests still pass
- **Summary of improvements**

If no changes:
- "No refactoring needed"
- Brief reasoning (e.g., "Implementation is minimal and already well-separated")
```

#### 使用钩子（hooks）提升 `tdd-integration` 技能触发几率
像 Claude Code 一样的编码代理并不保证触发技能，为了确保 TDD 得以以高几率运行，采用 hook 方案：

1. 修改 `.claude/settings.json`

采用 `UserPromptSubmit` 在每次用户提交提示词后， Claude 处理之前触发钩子脚本执行。

```json
{
    ...,
	"hooks": {
		"UserPromptSubmit": [
			{
				"matcher": "",
				"hooks": [
					{
						"type": "command",
						"command": "python \"$HOME/.claude/hooks/user-prompt-skill-eval.py\"",
						"timeout": 5
					}
				]
			}
		]
	}
}
```

2. hook 脚本（`$HOME/.claude/hooks/user-prompt-skill-eval.py`）
```python
#!/usr/bin/env python3
import sys

def main():
    # 读取并丢弃 stdin（用户原始 prompt）
    _ = sys.stdin.read()

    instruction = """
INSTRUCTION: MANDATORY SKILL ACTIVATION SEQUENCE

Step 1 - EVALUATE:
For each skill in <available_skills>, state:
[skill-name] - YES/NO - [reason]

Step 2 - ACTIVATE:
IF any skills are YES → Use Skill(skill-name) tool for EACH relevant skill NOW
IF no skills are YES → State "No skills needed" and proceed

Step 3 - IMPLEMENT:
Only after Step 2 is complete, proceed with implementation.

CRITICAL:
You MUST call Skill() tool in Step 2.
Do NOT skip to implementation.
"""

    # 输出到 stdout（Claude 会把它拼接进最终 prompt）
    sys.stdout.write(instruction.strip())

if __name__ == "__main__":
    main()
```

### TDD 的问题
尽管刚测试一个小的示例（石头-剪子-布），便可直观感受到其在速率、以及token花费上的问题。之后将继续使用与学习，以将进一步完善文章。


### 相关文章
- [Test Driven Development By Martin Fowler](https://martinfowler.com/bliki/TestDrivenDevelopment.html)
- [My Skill Makes Claude Code GREAT At TDD](https://www.aihero.dev/skill-test-driven-development-claude-code)
- [Forcing Claude Code to TDD: An Agentic Red-Green-Refactor Loop](https://alexop.dev/posts/custom-tdd-workflow-claude-code-vue/)
- [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide#linux)
