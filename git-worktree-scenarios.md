# Git Worktree 使用场景模拟

本文用 `h_test_worktree` 仓库举例，模拟 6 个常见场景。命令可直接复制执行（路径请按本机调整）。

**前置假设**

- 主仓库路径：`/Users/didi/code/aicode/h_test_worktree`
- 远程：`origin` → `github.com:Hrero/h_test_worktree.git`
- 已有分支：`main`、`feat/test`

---

## 场景 1：功能开发中，紧急修生产 Bug

### 背景

你在 `feat/test` 上改了一半，突然要在 `main` 上修一个线上问题。不想 `stash`，也不想 `checkout` 来回切。

### 目录规划

```text
aicode/
├── h_test_worktree/           # feat/test（继续功能开发）
└── h_test_worktree-hotfix/    # main（修 Bug）
```

### 操作步骤

```bash
cd /Users/didi/code/aicode/h_test_worktree

# 当前已在 feat/test，直接为 main 开 worktree
git fetch origin
git worktree add ../h_test_worktree-hotfix main

# 在 hotfix 目录修 Bug
cd ../h_test_worktree-hotfix
# ... 修改代码 ...
git add .
git commit -m "fix: 修复登录超时"
git push origin main

# 回到功能目录继续开发
cd ../h_test_worktree
# ... 无需 stash，feat/test 工作区原样保留 ...
```

### 结果

| 目录 | 分支 | 用途 |
|------|------|------|
| `h_test_worktree` | `feat/test` | 功能开发（未被打断） |
| `h_test_worktree-hotfix` | `main` | 紧急修复并已推送 |

### 收尾

```bash
cd /Users/didi/code/aicode/h_test_worktree
git worktree remove ../h_test_worktree-hotfix
```

---

## 场景 2：两个功能并行，各开一个 worktree

### 背景

同时做 `feat/test` 和 `feat/payment`，希望两个 Cursor 窗口各盯一个分支。

### 目录规划

```text
aicode/
├── h_test_worktree/              # feat/test
└── h_test_worktree-payment/      # feat/payment（新建）
```

### 操作步骤

```bash
cd /Users/didi/code/aicode/h_test_worktree

# 创建并检出 feat/payment
git fetch origin
git worktree add -b feat/payment ../h_test_worktree-payment

# 窗口 A：打开 h_test_worktree
# 窗口 B：打开 h_test_worktree-payment
```

在 payment 目录开发：

```bash
cd ../h_test_worktree-payment
echo "payment module" > payment.txt
git add payment.txt
git commit -m "feat: 添加 payment 模块"
git push -u origin feat/payment
```

### 注意

**同一分支不能出现在两个 worktree**。若报错：

```text
fatal: 'feat/test' is already checked out at '...'
```

说明该分支已被占用——当前目录先 `git checkout main`，或在新 worktree 使用其他分支。

---

## 场景 3：Code Review —— 并排对比两个分支

### 背景

Review 同事的 `feat/test`，同时对照 `main` 的差异，不想反复切换分支。

### 操作步骤

```bash
cd /Users/didi/code/aicode/h_test_worktree
git fetch origin

# 假设当前在 feat/test
git worktree add ../h_test_worktree-main main

# 终端 1：看 feature 改动
cd /Users/didi/code/aicode/h_test_worktree
git log main..HEAD --oneline

# 终端 2：对照 main 基线
cd ../h_test_worktree-main
git log -3 --oneline

# 跨目录 diff（路径按实际调整）
diff -ru ../h_test_worktree-main ./path/to/file \
        ../h_test_worktree/./path/to/file
```

### Cursor 用法

- 左侧窗口：`h_test_worktree`（feat/test）
- 右侧窗口：`h_test_worktree-main`（main）
- 用内置 diff 或搜索同名文件对照

---

## 场景 4：同一分支迁移到新目录（你遇到的报错）

### 背景

`h_test_worktree` 已在 `feat/test` 上，又想执行：

```bash
git worktree add ../h_test_worktree-v2 feat/test
```

报错：

```text
Preparing worktree (checking out 'feat/test')
fatal: 'feat/test' is already checked out at '/Users/didi/code/aicode/h_test_worktree'
```

### 原因

Git 默认：**一个分支只能被一个 worktree 检出**。

### 正确做法：先释放分支，再在新目录检出

```bash
cd /Users/didi/code/aicode/h_test_worktree

# 步骤 1：当前目录切走 feat/test
git checkout main

# 步骤 2：在新目录检出 feat/test
git worktree add ../h_test_worktree-v2 feat/test

# 步骤 3（可选）：若旧目录不再需要，可删除后只保留新目录
# git worktree remove .   # 需在非该 worktree 目录执行，且需先 cd 出去
```

### 迁移前后

| 阶段 | `h_test_worktree` | `h_test_worktree-v2` |
|------|-------------------|----------------------|
| 迁移前 | `feat/test` | （不存在） |
| 迁移后 | `main` | `feat/test` |

---

## 场景 5：发版前在旧 tag 上跑回归

### 背景

当前在 `main` 开发 v2.0，需要对照 v1.0 tag 跑测试，确认回归。

### 操作步骤

```bash
cd /Users/didi/code/aicode/h_test_worktree
git fetch --tags

# detached HEAD，不影响任何分支
git worktree add --detach ../h_test_worktree-v1.0 v1.0.0

cd ../h_test_worktree-v1.0
# 跑 v1.0 测试套件
# npm test / pytest / go test ./...

cd ../h_test_worktree
# main 上继续开发，互不影响
```

### 收尾

```bash
git worktree remove ../h_test_worktree-v1.0
```

### 提示

`--detach` 适合**只读/临时验证**，不适合长期 commit。若要基于旧版本开修复分支：

```bash
git worktree add -b hotfix/v1.0.1 ../h_test_worktree-v1-hotfix v1.0.0
```

---

## 场景 6：长生命周期分支 + 日常 main 开发

### 背景

`feat/test` 是长期分支（开发 2 周），日常还要在 `main` 上改文档、小修复。

### 目录规划

```text
aicode/
├── h_test_worktree/        # main（日常默认入口）
└── h_test_worktree-long/   # feat/test（长期功能）
```

### 初始化（一次性）

```bash
cd /Users/didi/code/aicode/h_test_worktree
git checkout main
git worktree add ../h_test_worktree-long feat/test
```

### 日常节奏

```bash
# 早上：main 上改 README
cd /Users/didi/code/aicode/h_test_worktree
git pull origin main
# ... 改文档、push ...

# 下午：long 目录继续功能
cd ../h_test_worktree-long
git pull origin feat/test
# ... 功能开发、push ...
```

### 合并功能完成后

```bash
cd /Users/didi/code/aicode/h_test_worktree
git merge feat/test          # 或在 GitHub 提 PR
git worktree remove ../h_test_worktree-long
git branch -d feat/test      # 可选
```

---

## 场景对照总览

| 场景 | 动机 | 关键命令 | 典型目录数 |
|------|------|----------|------------|
| 1 紧急修 Bug | 不打断当前功能分支 | `git worktree add ../xxx main` | 2 |
| 2 并行多功能 | 多窗口各盯一分支 | `git worktree add -b feat/xxx ../xxx` | 2+ |
| 3 Code Review | 并排对比分支 | 一个目录一分支 | 2 |
| 4 分支迁移 | 同分支换目录 | 先 `checkout` 其他分支，再 `worktree add` | 2 |
| 5 旧版回归 | 临时对照 tag | `git worktree add --detach ../xxx v1.0.0` | 2 |
| 6 长期分支 | main 与 feature 长期并存 | 主目录 main，附加目录 feature | 2 |

---

## 通用清理 checklist

完成某个场景后，建议执行：

```bash
cd /Users/didi/code/aicode/h_test_worktree

# 1. 查看所有 worktree
git worktree list

# 2. 删除不再需要的
git worktree remove ../h_test_worktree-hotfix

# 3. 若目录被手动删掉，清理注册
git worktree prune

# 4. 确认状态
git worktree list
git status
```

---

## 命名建议（本仓库）

```text
h_test_worktree/              # 主入口，建议固定 main
h_test_worktree-feat-test/    # feat/test
h_test_worktree-hotfix/       # 临时 hotfix
h_test_worktree-v1.0/         # 临时 tag 验证
```

后缀用 `-` 连接分支语义，比 `v2`、`copy` 更易辨认。

---

## 延伸阅读

- 基础教程：同目录 [`git-worktree.md`](./git-worktree.md)
- 官方文档：https://git-scm.com/docs/git-worktree
