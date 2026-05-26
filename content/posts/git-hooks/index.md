---
title: "Git Hooks"
date: 2026-05-25T22:20:36+08:00
draft: false
tags: ["git"]
comments: true
---
<center>
    <img src="./cover.png" alt="Git Hooks" title="Git Hooks" style="width: 100%;">
</center>
Git 提供了一种在特定重要操作发生时触发自定义脚本的机制，称为钩子（Hooks）。钩子本质上是 Git 仓库中 `.git/hooks` 目录下的可执行脚本，当仓库中发生特定事件时自动执行。这些脚本允许开发者自定义 Git 的内部行为，在开发生命周期的关键节点触发可自定义的操作。如 `commit` 前执行代码格式化、敏感信息检查或提交消息规范检查等。

`git init` 初始化 Git 仓库后，`.git/hooks` 目录下的官方钩子示例：
```text
.git/hooks (GIT_DIR!) 
$ ls
applypatch-msg.sample      pre-push.sample
commit-msg.sample          pre-rebase.sample
fsmonitor-watchman.sample  pre-receive.sample
post-update.sample         prepare-commit-msg.sample
pre-applypatch.sample      push-to-checkout.sample
pre-commit.sample          sendemail-validate.sample
pre-merge-commit.sample    update.sample
```

Git Hooks 根据执行位置可以分为客户端钩子（Client-side Hooks，本地钩子）与服务端钩子（Server-side Hooks，远端钩子）。

- 客户端钩子会在本地执行 Git 操作时触发，例如 `commit`、`merge`、`rebase`、`checkout`、`push` 等，常用于代码检查、格式化、测试与提交规范校验。

- 服务端钩子会在远程仓库接收到 `push` 请求时触发，通常运行于 Git 服务器（如 GitLab、Gitea、Gerrit）上，常用于权限控制、分支保护、CI/CD、自动部署与安全检查。

### 客户端钩子
客户端钩子可以进一步分为提交流程钩子、邮件流程钩子和其他钩子。

#### Commit Workflow Hooks

`git commit` 操作提交流程图
```
git commit
    │
    ├── pre-commit
    │
    ├── prepare-commit-msg
    │
    打开编辑器
    │
    ├── commit-msg
    │
    └── post-commit
```
- `pre-commit`: 该脚本在用户执行 `git commit` 后，Git 询问提交消息或生成提交对象之前执行。`pre-commit` 脚本不接受任何参数，如果以非零状态退出，则会中止提交。
```bash
#!/bin/sh

echo "🔍 Checking secrets..."

if git diff --cached --diff-filter=ACMR | grep -E '(API_KEY|SECRET_KEY|PASSWORD|TOKEN)'; then
  echo "❌ 检测到敏感信息，禁止提交"
  exit 1
fi

echo "✅ Passed"
```
- `prepare-commit-msg`: 该脚本在 commit message 编辑器打开前、默认消息创建之后运行。用于预填充 message ，接受1-3个参数，非零状态退出时会中止提交。
    - `$1`: 包含当前提交消息的文件路径（`.git/COMMIT_EDITMSG`）
    - `$2`: 提交类型（message: `-m` 或 `-F` 选项， template: `-t`选项，merge: 合并提交，squash: 压缩提交）
    - `$3`: 提交的 SHA-1 值（仅当使用 `-c`、`-C` 或 `--amend` 时提供）。

示例：从分支名提取 Issue 编号，自动前缀到提交消息中。
```bash
#!/usr/bin/env bash
# .git/hooks/prepare-commit-msg

COMMIT_MSG_FILE=$1
COMMIT_SOURCE=$2

# 仅处理普通提交，跳过 merge/squash/amend
if [ -n "$COMMIT_SOURCE" ]; then
  exit 0
fi

# 获取当前分支名，例如：feature/PROJ-123-login-page
BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null)

# 用正则提取形如 PROJ-123 的 Issue 编号
ISSUE=$(echo "$BRANCH" | grep -oE '[A-Z]+-[0-9]+')

if [ -n "$ISSUE" ]; then
  # 在原有内容前插入 Issue 编号
  CURRENT=$(cat "$COMMIT_MSG_FILE")
  echo "[$ISSUE] $CURRENT" > "$COMMIT_MSG_FILE"
fi
```
- `commit-msg`: 该脚本在用户输入 commit message，关闭编辑器后执行，常用于提醒开发者提交消息是否符合标准。该钩子接受一个参数，即开发者编写提交消息的临时文件的路径（`.git/COMMIT_EDITMSG`），非零状态退出时会中止提交。

示例：要求提交消息遵循 Conventional Commits 格式
```bash
#!/usr/bin/env bash
# .git/hooks/commit-msg

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Conventional Commits 正则
# 格式: type(scope): subject
PATTERN='^(feat|fix|docs|style|refactor|perf|test|chore|ci|build|revert)(\(.+\))?: .{1,72}$'

# 跳过 Merge commit
if echo "$COMMIT_MSG" | grep -qE '^Merge '; then
  exit 0
fi

# 逐行校验第一行（header）
HEADER=$(echo "$COMMIT_MSG" | head -n1)

if ! echo "$HEADER" | grep -qE "$PATTERN"; then
  echo ""
  echo "❌ 提交信息不符合 Conventional Commits 规范"
  echo ""
  echo "   正确格式: type(scope): subject"
  echo ""
  echo "   type 可选值:"
  echo "     feat     新功能"
  echo "     fix      修复 Bug"
  echo "     docs     文档变更"
  echo "     style    代码格式（不影响逻辑）"
  echo "     refactor 重构"
  echo "     perf     性能优化"
  echo "     test     测试相关"
  echo "     chore    构建/工具链"
  echo "     ci       CI 配置"
  echo "     revert   回滚"
  echo ""
  echo "   示例:"
  echo "     feat(auth): 添加 OAuth2 登录支持"
  echo "     fix(api): 修复用户列表分页错误"
  echo "     docs: 更新 README 安装说明"
  echo ""
  echo "   你的提交信息: \"$HEADER\""
  echo ""
  exit 1
fi

# 校验 subject 不能以大写字母开头（可选，按团队约定）
SUBJECT=$(echo "$HEADER" | sed 's/^[^:]*: //')
if echo "$SUBJECT" | grep -qE '^[A-Z]'; then
  echo "❌ subject 不应以大写字母开头: \"$SUBJECT\""
  exit 1
fi

echo "✅ Commit message check passed"
exit 0
```
- `post-commit`: 该脚本在提交过程完成后执行，不接受任何参数，主要用于通知。
示例：每次提交后，通过 Discord Webhook 通知
```bash
#!/usr/bin/env bash

WEBHOOK_URL="YOUR_DISCORD_WEBHOOK"

REPO=$(basename "$(git rev-parse --show-toplevel)")
BRANCH=$(git branch --show-current)

SHORT=$(git rev-parse --short HEAD)

AUTHOR=$(git log -1 --format="%an")
EMAIL=$(git log -1 --format="%ae")

MSG=$(git log -1 --format="%s" | sed 's/[[\]\\]/\\&/g')

TIME=$(date '+%Y-%m-%d %H:%M:%S')

STAT=$(git diff-tree --no-commit-id --stat -r HEAD | sed 's/[[\]\\]/\\&/g')

PATCH=$(git show --format="" --unified=1 HEAD | head -c 1500 | sed 's/[[\]\\]/\\&/g')

JSON=$(cat <<EOF
{
  "username": "GitBot",
  "embeds": [{
    "title": "📦 New Commit",
    "color": 5814783,
    "fields": [
      {
        "name": "Repository",
        "value": "$REPO",
        "inline": true
      },
      {
        "name": "Branch",
        "value": "$BRANCH",
        "inline": true
      },
      {
        "name": "Commit",
        "value": "$SHORT",
        "inline": true
      },
      {
        "name": "Author",
        "value": "$AUTHOR <$EMAIL>",
        "inline": false
      },
      {
        "name": "Message",
        "value": "$MSG",
        "inline": false
      },
      {
        "name": "Files Changed",
        "value": "\`\`\`$STAT\`\`\`",
        "inline": false
      },
      {
        "name": "Patch Preview",
        "value": "\`\`\`diff\n$PATCH\n\`\`\`",
        "inline": false
      }
    ],
    "footer": {
      "text": "$TIME"
    }
  }]
}
EOF
)

curl \
  -H "Content-Type: application/json" \
  -X POST \
  -d "$JSON" \
  "$WEBHOOK_URL" \
  > /dev/null 2>&1 &

exit 0
```
#### Email Workflow Hooks
在此之前，需要回顾一下补丁协作方式。开发者 A 使用 `git format-patch origin/main` 生成补丁文件 `*.patch`，之后将补丁发给维护者B（或许是 Linus），B 通过 `git am *.patch` 将邮件中的补丁应用于本地仓库。

邮件工作流涉及三个钩子，分别是 `applypatch-msg`, `pre-applypatch`, `post-applypatch`。 这三个钩子只在 `git am` 时触发，触发先后流程如下：
```
邮件 *.patch 文件
         │ git am *.patch
         ▼
┌─────────────────┐
│  applypatch-msg │  ← 钩子①：校验提交信息
└────────┬────────┘
         │ 通过
         ▼
┌─────────────────┐
│  pre-applypatch │  ← 钩子②：补丁打入后、提交前的检查
└────────┬────────┘
         │ 通过
         ▼
┌──────────────────┐
│ post-applypatch  │  ← 钩子③：提交完成后的通知
└──────────────────┘
```

- `applypatch-msg`: 该脚本在补丁应用前触发，接受一个参数（包含拟提交消息的临时文件名）。如果脚本以非零值退出，Git 将中止补丁操作。（相当于 `commit-msg` 钩子）

示例：补丁应用前，检查提交消息
```bash
#!/usr/bin/env bash
# .git/hooks/applypatch-msg

COMMIT_MSG_FILE=$1
HEADER=$(head -n1 "$COMMIT_MSG_FILE")

# 校验必须符合 Conventional Commits
PATTERN='^(feat|fix|docs|refactor|chore)(\(.+\))?: .+'
if ! echo "$HEADER" | grep -qE "$PATTERN"; then
  echo "❌ 补丁提交信息不符合规范: \"$HEADER\""
  exit 1   # 中止应用此补丁
fi

exit 0
```
- `pre-applypatch`: 该脚本在应用补丁后、提交之前运行，用来检查提交前的快照，如是否缺少某些内容或通过测试。（相当于 `pre-commit`）

示例：在提交前执行前端测试
```bash
#!/usr/bin/env bash
# .git/hooks/pre-applypatch

echo "🔍 运行测试，确保补丁不破坏现有功能..."

npm test
if [ $? -ne 0 ]; then
  echo "❌ 测试未通过，拒绝应用此补丁"
  exit 1   # 中止提交，但补丁内容已在工作区，可以手动检查
fi

exit 0
```
- `post-applypatch`: 在提交后执行，用来通知补丁作者或相关群组（相当于 `post-commit`）

#### 其他客户端钩子
- `pre-rebase`: 在执行任何 rebase 操作之前运行，用来禁止对已推送的提交进行变基。

- `post-rewrite`: 由替换提交的命令（如 `git commit --amend` 或 `git rebase`）触发，唯一的参数是触发重写的命令，并从 `stdin` 接收重写列表。

- `post-checkout`: 在成功执行 `git checkout` 后运行，用来为项目环境正确设置工作目录。

- `post-merge`: 在 `merge` 命令成功执行后运行，用来恢复工作树中 Git 无法跟踪的数据，如权限数据。

- `pre-push`: 在 `git push` 期间运行（在远程引用更新之后、任何对象传输之前），接受远程仓库的名称和位置作为参数，并通过 `stdin` 接收待更新引用的列表。可用来在推送发生前验证一组引用更新。

- `pre-auto-gc`: 执行 `git gc --auto` 后，Git 有时会自动执行垃圾回收，该钩子在垃圾回收发生之前调用，用来通知垃圾回收正在进行，或当前不适合垃圾回收。
```bash
#!/bin/sh

# 垃圾回收前提示
echo "⚠️ Git 即将开始自动垃圾回收"

exit 0
```

> Git 本质上是一个对象数据库，所有 commit，文件，历史，blob, tree 都会存进 `.git/objects/`。 在反复执行 git 操作的过程中，会产生很多无引用对象或临时对象，它们已无用但仍占用空间。
>
> Git GC（垃圾回收）：执行 `git gc`，进行垃圾回收（删除无用对象、压缩对象、优化仓库）。

### 服务器钩子
服务器钩子可用来为项目强制执行策略，这些钩子会在向服务器推送数据之前和之后运行。

`git push` 服务端处理流程：
```
客户端 git push
         │
         ▼
┌─────────────────┐
│  pre-receive    │  ← 钩子①：整个推送的前置校验（一次性）
└────────┬────────┘
         │ 通过
         ▼
┌─────────────────┐
│    update       │  ← 钩子②：逐引用校验（每个分支/标签各触发一次）
└────────┬────────┘
         │ 通过
         ▼
    引用更新完成
         │
         ▼
┌─────────────────┐
│  post-receive   │  ← 钩子③：推送完成后的通知与自动化
└─────────────────┘
```

- `pre-receive`: 在远程仓库接收到 `git push` 的数据后、任何引用更新之前运行。整个 push 操作只触发一次。脚本从 `stdin` 读取待更新的引用信息，每行格式为：`<old-sha> <new-sha> <ref-name>`。如果以非零状态退出，所有引用更新都会被拒绝，整个 push 失败。适合执行全局性策略，如禁止强制推送到受保护分支、限制推送文件大小等。

示例：禁止向 `main` 分支强制推送（`--force`），并限制单个推送的文件大小不超过 10MB
```bash
#!/usr/bin/env bash
# .git/hooks/pre-receive

ZERO="0000000000000000000000000000000000000000"

while read OLD_SHA NEW_SHA REF_NAME; do
  # 禁止强制推送到 main（old_sha 不为零且 new_sha 为零表示删除，忽略）
  if [ "$REF_NAME" = "refs/heads/main" ]; then
    # 检测强制推送：old_sha 非零，且 old_sha 不在 new_sha 的祖先链中
    if [ "$OLD_SHA" != "$ZERO" ] && [ "$NEW_SHA" != "$ZERO" ]; then
      MERGE_BASE=$(git merge-base "$OLD_SHA" "$NEW_SHA" 2>/dev/null)
      if [ "$MERGE_BASE" != "$OLD_SHA" ]; then
        echo "❌ 禁止强制推送到 main 分支"
        exit 1
      fi
    fi
  fi

  # 限制推送中单个文件大小不超过 10MB
  if [ "$OLD_SHA" = "$ZERO" ]; then
    # 新分支：裸仓库没有工作树，无法用 git diff 单 SHA
    FILES=$(git diff-tree --no-commit-id --name-only -r "$NEW_SHA" 2>/dev/null)
  else
    FILES=$(git diff --name-only "$OLD_SHA" "$NEW_SHA" 2>/dev/null)
  fi

  # 检查变更文件的大小
  for FILE in $FILES; do
    SIZE=$(git cat-file -s "${NEW_SHA}:${FILE}" 2>/dev/null)
    if [ -n "$SIZE" ] && [ "$SIZE" -gt 10485760 ]; then
      echo "❌ 文件 '$FILE' 大小为 $(($SIZE / 1048576))MB，超过 10MB 限制"
      exit 1
    fi
  done
done

exit 0
```

- `update`: 同样在引用更新之前运行，但与 `pre-receive` 不同的是，每个待更新的引用各触发一次。脚本接受三个参数：
    - `$1`: 引用名称（如 `refs/heads/main`）
    - `$2`: 旧 SHA-1 值
    - `$3`: 新 SHA-1 值

    如果以非零状态退出，仅拒绝该引用的更新，**其他引用不受影响**。适合执行针对特定分支或标签的策略，如标签命名规范、分支保护等。

示例：要求标签符合 SemVer 规范，并禁止删除 `release/*` 分支
```bash
#!/usr/bin/env bash
# .git/hooks/update

REF_NAME=$1
OLD_SHA=$2
NEW_SHA=$3

ZERO="0000000000000000000000000000000000000000"

case "$REF_NAME" in
  refs/tags/*)
    # 禁止删除标签
    if [ "$NEW_SHA" = "$ZERO" ]; then
      echo "❌ 禁止删除标签: $REF_NAME"
      exit 1
    fi

    # 标签必须符合 SemVer 格式（如 v1.2.3 或 v1.2.3-rc.1）
    TAG_NAME=$(echo "$REF_NAME" | sed 's|refs/tags/||')
    if ! echo "$TAG_NAME" | grep -qE '^v[0-9]+\.[0-9]+\.[0-9]+(-[a-zA-Z0-9.]+)?$'; then
      echo "❌ 标签 '$TAG_NAME' 不符合 SemVer 规范"
      echo "   正确格式: v<major>.<minor>.<patch>[-<prerelease>]"
      echo "   示例: v1.0.0, v2.1.3-rc.1"
      exit 1
    fi
    ;;

  refs/heads/release/*)
    # 禁止删除 release 分支
    if [ "$NEW_SHA" = "$ZERO" ]; then
      echo "❌ 禁止删除 release 分支: $REF_NAME"
      exit 1
    fi
    ;;
esac

exit 0
```

- `post-receive`: 在所有引用更新成功完成后运行，整个 push 操作只触发一次。与 `pre-receive` 类似，从 `stdin` 读取所有已更新的引用信息。由于此时引用已经更新完毕，脚本退出码不影响 push 结果。常用于触发 CI/CD 流水线、自动部署、发送通知等。

示例：推送后自动触发部署，并通过企业微信 Webhook 通知团队
```bash
#!/usr/bin/env bash
# .git/hooks/post-receive

while read OLD_SHA NEW_SHA REF_NAME; do
  BRANCH=$(echo "$REF_NAME" | sed 's|refs/heads/||')
  ZERO="0000000000000000000000000000000000000000"

  # 跳过分支删除操作
  if [ "$NEW_SHA" = "$ZERO" ]; then
    continue
  fi

  # main 分支 → 触发生产环境部署
  if [ "$BRANCH" = "main" ]; then
    echo "🚀 触发生产环境部署..."
    # 异步触发部署脚本，不阻塞 push
    nohup /opt/deploy/production.sh > /dev/null 2>&1 &
  fi

  # staging 分支 → 触发预发布环境部署
  if [ "$BRANCH" = "staging" ]; then
    echo "🔧 触发预发布环境部署..."
    nohup /opt/deploy/staging.sh > /dev/null 2>&1 &
  fi

  # 构建通知消息
  SHORT=$(git rev-parse --short "$NEW_SHA")
  AUTHOR=$(git log -1 --format='%an' "$NEW_SHA")
  MSG=$(git log -1 --format='%s' "$NEW_SHA")
  REPO=$(basename "$(git rev-parse --git-dir)" .git)
  TIME=$(date '+%Y-%m-%d %H:%M:%S')

  # 企业微信 Webhook 通知
  WEBHOOK_URL="YOUR_WECHAT_WEBHOOK_URL"

  JSON=$(cat <<EOF
{"msgtype":"markdown","markdown":{"content":"## 📦 代码推送通知\n> **仓库**: $REPO\n> **分支**: $BRANCH\n> **提交**: $SHORT\n> **作者**: $AUTHOR\n> **消息**: $MSG\n> **时间**: $TIME"}}
EOF
  )

  curl -s -X POST "$WEBHOOK_URL" \
    -H 'Content-Type: application/json' \
    -d "$JSON" \
    > /dev/null 2>&1 &
done

exit 0
```

`pre-receive` vs `update` 的选择：

| 特性 | `pre-receive` | `update` |
|------|--------------|---------|
| 触发次数 | 整个 push 一次 | 每个引用各一次 |
| 失败影响 | 拒绝整个 push | 仅拒绝当前引用 |
| 引用信息 | 从 stdin 读取 | 通过参数传入 |
| 适用场景 | 全局策略、跨引用检查 | 单分支/标签独立策略 |

钩子的类型由文件名决定，当我们希望启用某种钩子时，只需在 `.git/hooks` 下存放对应名称的可执行脚本（无扩展）即可，对脚本本身采用的编程语言没有限制。

由于 `.git/` 目录没有纳入版本控制，钩子不会提交至远端，团队间共享钩子的一个简单方法是使用实际项目目录存储钩子，并在 `.git/hooks` 创建钩子的符号链接或者拷贝；其他方案如 [Husky](https://github.com/typicode/husky)