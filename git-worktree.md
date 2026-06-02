# Git Worktree 教程

Git **worktree**（工作树）让你从**同一个仓库**里，同时检出**多个分支**到**不同目录**，各自独立编辑，共享同一份 `.git` 对象数据库。

> **说明**：若目录是独立 `git init` 的仓库（自带完整 `.git` 文件夹），则不是 worktree。真正的 worktree 目录里 `.git` 是一个指向主仓库的文本文件。

---

## 一、解决什么问题？

| 场景 | 传统做法 | Worktree |
|------|----------|----------|
| 在 `feature` 上开发，同时要修 `main` 的 bug | `git stash` 或再 `git clone` 一份 | 再开一个目录，检出 `main` |
| 同时对比两个分支的代码/跑测试 | 来回 `git checkout` | 两个目录各盯一个分支 |
| 本地跑多个长期分支（如 `dev` + `hotfix`） | 多个 clone，占磁盘、同步麻烦 | 共享对象库，更省空间 |

核心：**一个仓库，多个工作目录，每个目录固定在一个分支上。**

---

## 二、核心概念

```
my-repo/                 ← 主 worktree（第一个检出目录）
├── .git/                ← 真正的 Git 目录
├── src/
└── ...

my-repo-feature/         ← 附加 worktree（链接到同一仓库）
├── .git                 ← 文件，指向主仓库的 .git/worktrees/...
└── src/                 ← 这里是 feature 分支的文件
```

- **主 worktree**：`git clone` 或 `git init` 后第一次检出的目录。
- **附加 worktree**：用 `git worktree add` 创建的额外目录。
- 所有 worktree **共享提交、分支、远程**，但**工作区文件互不覆盖**。

---

## 三、常用命令

### 1. 查看所有 worktree

```bash
git worktree list
```

示例输出：

```text
/path/to/my-repo        abc1234 [main]
/path/to/my-repo-hotfix def5678 [hotfix/login]
```

### 2. 新建 worktree（最常用）

```bash
# 基于已有远程/本地分支
git worktree add ../my-repo-feature feature/login

# 基于当前 HEAD 新建分支并检出
git worktree add -b feature/new-ui ../my-repo-ui

# 检出某个 commit（detached HEAD，适合临时查看）
git worktree add --detach ../my-repo-old v1.2.0
```

路径习惯：放在主仓库**同级**或**子目录**，例如：

```text
projects/
├── my-app/              # 主 worktree，main
└── my-app-feature/      # 附加 worktree，feature/xxx
```

### 3. 删除 worktree

```bash
# 确认目录内改动已提交或不需要后
git worktree remove ../my-repo-feature

# 目录已被手动删除时，清理注册信息
git worktree prune
```

### 4. 移动 worktree 目录

```bash
git worktree move ../old-path ../new-path
```

---

## 四、完整实战流程

假设主项目在 `/path/to/my-app`，当前在 `main` 上开发。

**场景**：在 `feature/payment` 上开发，同时紧急修 `main`。

```bash
cd /path/to/my-app

# 1. 确保分支存在
git fetch origin
git branch feature/payment origin/feature/payment 2>/dev/null || git checkout -b feature/payment

# 2. 新建 worktree
git worktree add ../my-app-payment feature/payment

# 3. 在另一个目录修 main
git worktree add ../my-app-hotfix main
cd ../my-app-hotfix
# ... 修 bug、commit、push ...

# 4. 回到 feature 目录继续开发
cd ../my-app-payment
# ... 正常开发 ...

# 5. feature 合并完后删除 worktree
cd /path/to/my-app
git worktree remove ../my-app-payment
git branch -d feature/payment   # 可选：删除本地分支
```

---

## 五、从零创建 worktree 目录

若希望某个目录（例如 `h_test_worktree`）**作为 worktree** 挂在主仓库下，应在主仓库中执行：

```bash
cd /path/to/主仓库
git worktree add /path/to/h_test_worktree -b h-test
```

此时 `h_test_worktree/.git` 是指向主仓库的**指针文件**，而不是完整的 `.git` 目录。

若已在目标目录执行过 `git init`，需先移除该独立仓库的 `.git`，或换一个空目录再 `git worktree add`。

---

## 六、Worktree vs 再 Clone 一份

| | Worktree | 再 `git clone` |
|--|----------|----------------|
| 磁盘 | 共享对象，更省 | 完整复制一份 `.git` |
| 分支/远程 | 完全同步 | 需各自 `fetch` |
| 隔离性 | 工作区隔离，仓库一体 | 完全隔离 |
| 适合 | 同项目多分支并行 | 不同权限、完全不同项目 |

---

## 七、注意事项与常见问题

### 1. 同一分支不能同时在两个 worktree 检出

会报错：

```text
fatal: 'xxx' is already checked out at '...'
```

**解决**：在其中一个 worktree 切换到其他分支，或删除多余的 worktree。

### 2. 删除目录请用官方命令

不要只 `rm -rf` 目录，应使用：

```bash
git worktree remove <path>
```

若已手动删除，执行 `git worktree prune` 清理失效记录。

### 3. 子模块（submodule）

每个 worktree 需各自初始化：

```bash
git submodule update --init --recursive
```

### 4. IDE / Cursor

每个 worktree 作为独立项目根目录打开（不同窗口）。

### 5. 推送

任意 worktree 内 `git push` / `git pull` 效果相同，共享同一套分支与远程配置。

### 6. 推送失败（non-fast-forward）

某个 worktree 推送被拒时，在该目录执行：

```bash
git pull --rebase origin <branch>
git push
```

---

## 八、速查表

```bash
git worktree list                    # 列出所有 worktree
git worktree add <path> <branch>     # 基于已有分支新建
git worktree add -b <branch> <path>  # 新建分支并检出
git worktree remove <path>           # 删除 worktree
git worktree prune                   # 清理失效记录
git worktree move <old> <new>        # 移动目录
```

---

## 九、推荐目录命名

```text
my-app/              # main
my-app-feat-auth/    # feature/auth
my-app-fix-123/      # fix/issue-123
```

在目录名中体现分支或用途，便于长期维护时识别。

---

## 参考

- [Git 官方文档：git-worktree](https://git-scm.com/docs/git-worktree)
- [Git 官方书籍：Worktrees](https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging#_worktrees)
