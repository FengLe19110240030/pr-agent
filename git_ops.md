# Git 操作速查

本文档针对当前仓库：

```bash
cd /Users/fengle/code/PycharmProjects/pr-agent
```

当前常用远程仓库：

```bash
git remote -v
```

预期会看到：

```text
fengle  https://github.com/FengLe19110240030/pr-agent.git (fetch)
fengle  https://github.com/FengLe19110240030/pr-agent.git (push)
origin  https://github.com/The-PR-Agent/pr-agent.git (fetch)
origin  https://github.com/The-PR-Agent/pr-agent.git (push)
```
## 1. 首次拉取远程代码并配置 GitHub Classic Token

适用场景：使用 HTTPS 远程地址从 GitHub 拉代码，并把 GitHub personal access token classic 缓存在本机，后续 `git pull` 和 `git push` 不用每次重新输入。

重要：不要把 token 写进 `git_ops.md`、代码、提交记录、shell 历史或 remote URL。GitHub 官方文档说明，personal access token 可以在命令行里代替密码使用；输入用户名和密码时，用户名填 GitHub 用户名，密码位置填 token。

### 1.1 创建 classic token

在浏览器打开 GitHub：

```text
头像 -> Settings -> Developer settings -> Personal access tokens -> Tokens (classic) -> Generate new token -> Generate new token (classic)
```

建议填写：

```text
Note: local-git-pr-agent
Expiration: 选择一个明确过期时间，例如 90 days 或 1 year
Scopes:
  public_repo  # 只推送公开仓库时通常够用
  repo         # 需要访问私有仓库时使用，权限更大
  workflow     # 如果要修改并推送 .github/workflows/ 下的工作流文件，再勾选
```

生成后只会显示一次，复制后放到临时的密码管理器或系统剪贴板里。不要保存到仓库文件。

官方参考：

```text
https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
https://docs.github.com/en/get-started/git-basics/caching-your-github-credentials-in-git
```

### 1.2 确认远程地址使用 HTTPS

```bash
git remote -v
```

如果远程地址不是 HTTPS，可以改成 HTTPS。这里以个人 fork `fengle` 为例：

```bash
git remote set-url fengle https://github.com/FengLe19110240030/pr-agent.git
```

如果还没有克隆仓库，可以直接从远程拉取一份：

```bash
git clone https://github.com/FengLe19110240030/pr-agent.git
cd pr-agent
```

如果要保留上游仓库作为只读同步源：

```bash
git remote add origin https://github.com/The-PR-Agent/pr-agent.git
git remote -v
```

### 1.3 配置凭据缓存

macOS 推荐使用系统 Keychain。先确认是否可用：

```bash
git credential-osxkeychain
```

如果输出用法说明，说明可用，然后配置：

```bash
git config --global credential.helper osxkeychain
```

也可以使用 Git Credential Manager。安装完成后，GitHub 凭据会保存到系统凭据管理器里：

```bash
brew install --cask git-credential-manager
```

### 1.4 第一次拉取远程代码

进入仓库目录：

```bash
cd /Users/fengle/code/PycharmProjects/pr-agent
```

查看远程和分支：

```bash
git remote -v
git status --short --branch
```

从个人 fork 拉取 `main`：

```bash
git fetch fengle --prune
git switch main
git pull --ff-only fengle main
```

如果要先同步上游官方仓库，再推回自己的 fork：

```bash
git fetch origin --prune
git switch main
git merge --ff-only origin/main
git push fengle main
```

### 1.5 第一次 push 时输入 token

推送到个人 fork：

```bash
git push fengle main
```

如果 Git 提示输入用户名和密码：

```text
Username for 'https://github.com': FengLe19110240030
Password for 'https://FengLe19110240030@github.com': 粘贴 classic token
```

输入成功后，凭据 helper 会把 token 缓存在本机。后续通常可以直接运行：

```bash
git pull
git push
```

### 1.6 更新或清除旧 token

如果 token 过期、被 revoke，或者输错后被缓存，可以清除 GitHub 的旧凭据，再重新 `git pull` 或 `git push` 触发输入。

macOS Keychain：

```bash
git credential-osxkeychain erase
host=github.com
protocol=https

```

注意最后需要输入一个空行来提交 erase 请求。

也可以打开“钥匙串访问”，搜索 `github.com`，删除对应的 Internet password 条目。

## 1. 查看仓库状态

```bash
cd /Users/fengle/code/PycharmProjects/pr-agent
git status
```

简洁格式：

```bash
git status --short --branch
```

## 2. 查看当前分支

```bash
git branch --show-current
```

查看所有本地分支：

```bash
git branch
```

查看本地和远程分支：

```bash
git branch -a
```

## 3. 查看提交历史

```bash
git log --oneline
```

查看最近 5 条提交：

```bash
git log --oneline -5
```

查看图形化历史：

```bash
git log --oneline --graph --decorate --all -20
```

## 4. 拉取远程更新

当前 `main` 分支跟踪的是 `fengle/main`，可以直接运行：

```bash
git pull
```

也可以明确指定远程和分支：

```bash
git pull fengle main
```

只获取远程更新，不合并：

```bash
git fetch fengle
```

## 5. 查看文件改动

查看工作区未暂存改动：

```bash
git diff
```

查看某个文件的改动：

```bash
git diff git_ops.md
```

查看已暂存、即将提交的改动：

```bash
git diff --cached
```

## 6. 提交修改

提交前先检查状态：

```bash
git status --short --branch
```

提交前检查是否误写入 GitHub token：

```bash
rg "github_pat_|ghp_" .
```

暂存单个文件：

```bash
git add git_ops.md
```

暂存全部修改：

```bash
git add .
```

提交：

```bash
git commit -m "docs: update git ops"
```

## 7. 推送到远程仓库

当前分支已经跟踪 `fengle/main`，可以直接运行：

```bash
git push
```

也可以明确指定远程和分支：

```bash
git push fengle main
```

如果遇到 HTTP/2 传输问题，可以使用：

```bash
git -c http.version=HTTP/1.1 -c http.postBuffer=524288000 push
```

## 8. 完整日常流程

修改文件后，按这个流程提交并推送：

```bash
cd /Users/fengle/code/PycharmProjects/pr-agent
git status --short --branch
git pull
git diff
rg "github_pat_|ghp_" .
git add .
git diff --cached
git commit -m "docs: update git ops"
git push
git status --short --branch
```

## 9. 修改最近一次提交

如果提交后发现漏改了文件：

```bash
git add .
git commit --amend --no-edit
```

如果只想修改最近一次提交信息：

```bash
git commit --amend -m "docs: update git ops"
```

如果最近一次提交还没有推送，修改后直接推送：

```bash
git push
```

如果最近一次提交已经推送过，修改历史后需要谨慎使用：

```bash
git push --force-with-lease
```

## 10. 撤销修改

撤销某个文件的工作区修改：

```bash
git restore git_ops.md
```

取消暂存某个文件：

```bash
git restore --staged git_ops.md
```

撤销所有未暂存的工作区修改：

```bash
git restore .
```

## 11. 新建和切换分支

新建并切换到功能分支：

```bash
git switch -c feature/git-ops
```

切回 `main`：

```bash
git switch main
```

推送新分支到远程：

```bash
git push -u fengle feature/git-ops
```

## 12. 合并分支

先切回 `main`：

```bash
git switch main
```

拉取最新代码：

```bash
git pull
```

合并功能分支：

```bash
git merge feature/git-ops
```

推送合并结果：

```bash
git push
```

## 13. 删除分支

删除本地分支：

```bash
git branch -d feature/git-ops
```

删除远程分支：

```bash
git push fengle --delete feature/git-ops
```

## 14. 查看远程 main 是否同步

查看本地 `main` 提交：

```bash
git rev-parse main
```

查看远程 `fengle/main` 提交：

```bash
git ls-remote fengle refs/heads/main
```

如果两个提交 SHA 一样，说明本地和远程同步。

## 15. 处理 Push Protection 拦截

如果推送时报错：

```text
GH013: Repository rule violations found
Push cannot contain secrets
```

先在当前文件中搜索 token：

```bash
rg "github_pat_|ghp_" .
```

如果当前文件已经没有 token，但 GitHub 仍然拒绝，说明 token 可能存在于尚未推送的提交历史中。查看待推送提交：

```bash
git log --oneline fengle/main..HEAD
```

搜索某个提交里的 token：

```bash
git grep -n "ghp_" HEAD
git grep -n "github_pat_" HEAD
```

如果只是最近几个未推送提交有问题，可以把它们压成一个干净提交：

```bash
git reset --soft fengle/main
rg "github_pat_|ghp_" .
git add .
git commit -m "docs: update git ops"
git push
```

重要：泄露过的 token 必须到 GitHub 后台 revoke，不能继续使用。
