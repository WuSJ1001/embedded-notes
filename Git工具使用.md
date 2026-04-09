## 结构预览
- Git 基础操作：常用命令速览与使用要点。
- 创建与连接远程仓库：从初始化到推送的完整流程。
- SSH 绑定：生成密钥并配置 GitHub。
- Obsidian 集成：安装插件、配置远程仓库与自动同步。
## Git 基础操作

- **查看状态**：`git status`，了解当前工作区与暂存区的差异。
- **跟踪/暂存**：`git add <file>`，批量可用 `git add .`；查看差异用 `git diff`，已暂存差异用 `git diff --staged`。
- **提交**：`git commit -m "说明"`；已跟踪文件可用 `git commit -am "说明"` 跳过 `add`。
- **删除或取消跟踪**：
  - 完全移除：`git rm <file>`
  - 仅从索引移除保留物理文件：`git rm -r --cached <path>`
- **移动/重命名**：`git mv <old> <new>`
- **忽略文件**：在 `.gitignore` 中列出文件或目录（目录名后加 `/`），注意保存为 UTF-8 编码。
- **查看历史**：`git log`
- **分支操作**：
  - 列表：`git branch`
  - 创建：`git branch <name>`
  - 删除：`git branch -d <name>`
  - 切换：`git checkout <name>`
  - 合并：`git merge <name>`（把 `<name>` 合并入当前分支）
- **标签**：`git tag <tag-name>`

## 创建与连接远程仓库

- **本地准备**
  1. 创建项目文件夹。
  2. 在文件夹内运行 `git init` 初始化仓库。
- **添加远程地址**
  - SSH：
  ```
  git remote add origin git@github.com:用户名/仓库名.git
  ```
- HTTPS：
- ```
  git remote add origin https://github.com/用户名/仓库名.git
  ```
  
- **同步远程内容**
  - 首次拉取：`git pull origin 分支名`
  - 检查工作区：`git status`，必要时 `git add` 暂存。
- **提交与推送**
  1. `git commit -m "备注"` 生成本地提交。
  2. 首次推送：`git push -u origin 本地分支` 绑定上游；之后可直接 `git push`。
- **常用 push 参数**
  - `-u` / `--set-upstream`：绑定远程分支。
  - `-f` / `--force`：强制覆盖（慎用，建议改用 `--force-with-lease`）。
  - `-d` / `--delete`：删除远程分支。
  - `--tags`：推送标签。
  - `--no-verify`：跳过本地钩子检查（仅在确认安全时使用）。
- **常见状态提示**
  - `Untracked files`：使用 `git add .` 或逐个添加。
  - `Changes not staged for commit`：使用 `git commit -am "备注"` 或手动 `git add` 后提交。

## 配置 SSH 访问

- **生成密钥**：运行 `ssh-keygen -t rsa`，一路回车生成 `id_rsa`（私钥）与 `id_rsa.pub`（公钥），默认位于：
  - Linux / macOS：`~/.ssh/`
  - Windows：`C:\Users\<用户名>\.ssh\`
- **添加到 GitHub**
  1. 在 GitHub 个人设置中打开 **SSH and GPG keys**。
  2. 点击 **New SSH key**，粘贴 `id_rsa.pub` 内容并保存。
- **验证**：执行 `ssh -T git@github.com`，出现欢迎信息即表示绑定成功。

## Obsidian 与 Git 集成

- **安装插件**
  1. 在 Obsidian 中依次进入 *设置 → 第三方插件 → 浏览*。
  2. 搜索并安装 **Git** 插件，启用后在设置中配置 `Custom Git Binary Path`，指向本机 `git.exe`。
- **理解仓库概念**
  - Obsidian Vault：笔记根目录。
  - Git Repository：Git 用于跟踪的仓库。目标是把 Vault 变成 Git 仓库。
- **初始化仓库**
  - 在命令面板执行 **Git: Initialize a new repository**，Vault 内会生成 `.git` 文件夹。
- **配置远程（以 Gitee 为例）**
  1. 在 Gitee 创建私有仓库（如 `my-obsidian-notes`）。
  2. 在 Obsidian 命令面板运行 **Git: Edit remote**，远程名填 `origin`，URL 粘贴 Gitee 仓库的 HTTPS 地址。
- **同步与自动化**
- 打开 **Git: Open source control view** 查看待提交文件，按需 `Commit`、`Pull`、`Push`。
- 插件设置里可开启自动提交、自动推送或按时间间隔同步，让 Obsidian 启动时自动拉取/推送。

## 参考

- [还不会使用 GitHub？ GitHub 教程来了！万字图文详解 - 知乎](https://zhuanlan.zhihu.com/p/369486197)
