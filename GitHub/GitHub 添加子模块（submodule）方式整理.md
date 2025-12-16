> 目标：把一个 Git 仓库作为“子项目”挂到另一个仓库里，主仓库只记录子仓库的**提交指针**（commit hash），而不是把子仓库代码直接拷贝进来。

---

## 1. 常用场景与特点

### 适合用 submodule 的场景
- 多个项目共享同一套库/组件（且希望独立版本演进）
- 主仓库需要固定依赖某个子仓库的具体版本（精确到 commit）
- 依赖仓库不希望被主仓库“复制”进历史

### 关键特点（一定要理解）
- 主仓库不会跟踪子模块内部文件变更，只记录：
  - `.gitmodules`（子模块配置）
  - 子模块目录对应的 **gitlink**（指向子模块某个 commit）
- 子模块默认不是自动更新的：拉主仓库代码后，子模块可能还停留在旧 commit，需要单独更新

---

## 2. 添加子模块（最常用）

### 命令
```bash
# 在主仓库根目录执行
git submodule add <子仓库URL> <路径/目录名>
```

### 示例
```bash
git submodule add https://github.com/owner/lib-a.git libs/lib-a
```

### 提交到主仓库
```bash
git status
# 会看到：新增 .gitmodules + 新增子模块目录（gitlink）
git add .gitmodules libs/lib-a
git commit -m "Add lib-a as submodule"
git push
```

---

## 3. 克隆包含子模块的仓库

### 方式 A：一步到位（推荐）
```bash
git clone --recurse-submodules <主仓库URL>
```

### 方式 B：先克隆再初始化/更新
```bash
git clone <主仓库URL>
cd <主仓库目录>

git submodule init
git submodule update
```

### 方式 C：同时拉取子模块的远端更新（常用组合）
```bash
git submodule update --init --recursive
```

---

## 4. 更新子模块指针（主仓库想“升级依赖”）

> 意味着：让主仓库记录子模块的新 commit。

### 方式 A：进入子模块目录，拉取并切到目标分支/提交
```bash
cd libs/lib-a
git fetch
git checkout main
git pull
```

然后回到主仓库提交“指针变化”：
```bash
cd ../..
git status
# 会看到子模块变更（new commits）
git add libs/lib-a
git commit -m "Bump lib-a submodule"
git push
```

### 方式 B：使用主仓库命令更新（不进目录）
```bash
# 更新子模块到主仓库记录的 commit（不是最新）
git submodule update

# 拉子模块远端，并更新到远端分支最新（需要 --remote）
git submodule update --remote
```

> `--remote` 会根据子模块配置的分支（或默认）去更新到最新提交，随后仍需要在主仓库 `git add <submodule>` 并提交。

---

## 5. 指定子模块跟踪的分支（可选）

### 让子模块默认跟踪某个分支
```bash
git submodule add -b develop <子仓库URL> libs/lib-a
```

或者为已存在的子模块设置分支（写入 `.gitmodules`）：
```bash
git config -f .gitmodules submodule.libs/lib-a.branch develop
git add .gitmodules
git commit -m "Configure lib-a submodule branch to develop"
```

之后用：
```bash
git submodule update --remote
```

---

## 6. 子模块递归（子模块里还有子模块）

### 克隆/更新递归处理
```bash
git clone --recurse-submodules <主仓库URL>

# 或者后续执行
git submodule update --init --recursive
```

---

## 7. 删除子模块（容易踩坑，建议按步骤）

> 不同 Git 版本与历史操作会有差异，下面是相对稳妥的一套流程。

假设子模块路径是 `libs/lib-a`

### 1）从索引中移除子模块（保留工作区文件可选）
```bash
git submodule deinit -f -- libs/lib-a
git rm -f libs/lib-a
```

### 2）清理配置与残留目录
```bash
rm -rf .git/modules/libs/lib-a
```

### 3）提交
```bash
git commit -m "Remove lib-a submodule"
git push
```

> 如果你想保留子模块目录中的文件作为普通代码（不再是 submodule），可以先备份/复制出来再执行 `git rm`，最后把文件以普通方式 `git add` 回来。

---

## 8. 日常协作建议（团队约定）

- **新人克隆**统一要求：
  - `git clone --recurse-submodules ...`
  - 或 clone 后执行 `git submodule update --init --recursive`
- CI/CD 中拉代码建议添加：
  - `git submodule update --init --recursive`
- 升级子模块版本时要记得：
  - **子模块目录 pull 之后**，必须在主仓库 `git add 子模块路径` 并提交（记录新指针）
- 尽量避免在子模块目录里“直接改代码但不提交/不推送”，否则主仓库只能记录到某个 commit，协作者无法复现你本地的未提交修改

---

## 9. 常见问题速查（FAQ）

### Q1：我在主仓库看到子模块是 “(new commits)” 怎么提交？
```bash
git add libs/lib-a
git commit -m "Update submodule pointer"
```

### Q2：为什么我 clone 后子模块目录是空的？
因为 submodule 默认不会自动拉取内容，需要：
```bash
git submodule update --init
```
或 clone 时用 `--recurse-submodules`

### Q3：子模块处于 detached HEAD 正常吗？
正常：主仓库记录的是某个 commit，子模块检出该 commit 通常就是 detached HEAD。
如需在子模块上开发，建议切到分支：
```bash
cd libs/lib-a
git checkout main
```

---

## 10. 一套推荐的“最小指令集”（记住这几个就够用）

- 添加子模块：
  - `git submodule add URL path && git commit`
- 克隆并拉子模块：
  - `git clone --recurse-submodules URL`
- 现有仓库拉子模块：
  - `git submodule update --init --recursive`
- 更新到子模块远端最新：
  - `git submodule update --remote && git add <submodule> && git commit`

---