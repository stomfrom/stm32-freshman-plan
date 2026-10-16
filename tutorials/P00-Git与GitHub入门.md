# P00 手把手教程：Git 与 GitHub 入门

> 配套任务卡：`tasks/P00-Git与GitHub入门.md`
> 环境：**Windows**（命令示例以 Windows 命令行/PowerShell 为准，Git 命令本身跨平台一致）
> 这一课**不需要开发板**，只需要一台电脑。

---

## 目录

| 任务点 | 教程 |
|---|---|
| A1 说清 Git 与网盘的不同 | [跳转](#p00-a1) |
| A2 安装 Git | [跳转](#p00-a2) |
| A3 配置用户名与邮箱 | [跳转](#p00-a3) |
| A4 建本地仓库 | [跳转](#p00-a4) |
| A5 第一次提交 | [跳转](#p00-a5) |
| A6 观察"已修改未提交" | [跳转](#p00-a6) |
| A7 注册并建远程仓库 | [跳转](#p00-a7) |
| A8 连接远程并推送 | [跳转](#p00-a8) |
| A9 写 .gitignore | [跳转](#p00-a9) |
| B1 分支与合并 | [跳转](#p00-b1) |
| B2 改坏了怎么救回来 | [跳转](#p00-b2) |
| B3 clone 与下载压缩包的区别 | [跳转](#p00-b3) |
| B4 在网页上看提交历史 | [跳转](#p00-b4) |
| B5 写一份能看懂的 README | [跳转](#p00-b5) |
| B6 提交信息的重要性 | [跳转](#p00-b6) |

---

<a id="p00-a0"></a>
## 0 · 开工前的准备

### 0.1 三个区域：理解 Git 的关键

Git 把文件分成三个区域，**理解这张图，就理解了 Git 一半**：

```
┌──────────────┐   git add    ┌──────────────┐  git commit  ┌──────────────┐
│   工作区     │ ───────────> │   暂存区     │ ───────────> │   本地仓库   │
│ (你正在改的  │              │ (准备提交的   │              │ (已存档的    │
│  文件)       │ <─────────── │  改动)       │ <─────────── │  历史版本)   │
└──────────────┘   git restore└──────────────┘              └──────┬───────┘
                                                                    │ git push
                                                                    v
                                                            ┌──────────────┐
                                                            │  GitHub 远程 │
                                                            └──────────────┘
```

**`git status`** 就是让你随时看清"我的文件现在在哪一层"。

### 0.2 你会用到的命令（先混个眼熟）

| 命令 | 作用 |
|---|---|
| `git status` | 现在什么状态 |
| `git add <文件>` | 把改动放进暂存区 |
| `git commit -m "说明"` | 存成一个版本 |
| `git log` | 看历史 |
| `git push` | 推到 GitHub |

**这五条覆盖日常 90% 的用法。**

---

<a id="p00-a1"></a>
## A1 · 说清 Git 与网盘的不同

**任务**：用自己的话写一段不超过 100 字的说明：Git 与网盘（或 QQ 传文件）有什么不同。写进下面第 8 条的仓库 README 里。

### 知识点

**核心差异在"记录了什么"**：

| | 网盘 / QQ 传文件 | Git |
|---|---|---|
| 存的是 | 文件**最终的样子** | 文件**每一次变化**（谁、何时、改了哪几行、为什么） |
| 想回退 | 靠文件名区分（`最终版`、`最终版2`、`真最终版`） | 一条命令回到任意历史版本 |
| 想知道"改了什么" | 靠人脑记 | 能精确对比任意两个版本的差异 |
| 多人协作 | 互相覆盖，靠约定 | 自动合并，冲突会明确提示 |

**一句话**：网盘存"**结果**"，Git 存"**过程**"。

### 操作步骤

写一段不超过 100 字的说明（不用华丽，说清"过程 vs 结果"这一点就行）。示例：

```
Git 记录的是文件的每一次变化——谁在什么时候改了什么、为什么改，
可以随时回到任意一个历史版本。网盘只保存文件最后的样子，
想看改了什么只能靠人比。Git 是"时间机器"，网盘是"储物柜"。
```

先记下来，A8 会把它放进仓库 README。

### 验证方法

- 说明里区分出了"记录过程"和"保存结果"这两个点
- 字数 ≤100
- 不是为了交差复制粘贴——评审时会被要求口头解释

---

<a id="p00-a2"></a>
## A2 · 安装 Git

**任务**：在电脑上安装 Git，装完打开命令行执行 `git --version`，把输出的版本号截图。

### 操作步骤

1. 打开官网下载页：`https://git-scm.com/download/win`，下载 **64-bit Git for Windows Setup**
2. 双击安装，**一路点 Next 即可**，只有一个地方注意：

   | 安装页 | 选择 |
   |---|---|
   | `Default editor` | 选 **Notepad** 或你常用的编辑器（默认 Vim 对新手不友好） |
   | `Adjusting your PATH` | 选 **Git from the command line and also from 3rd-party software**（默认项） |
   | 其余 | 全部默认 |

3. 装完后**新开**一个命令行窗口（重要：旧窗口读不到新装的 PATH）
4. 执行：

```powershell
git --version
```

**预期输出**（版本号会不同）：

```
git version 2.44.0.windows.1
```

5. 截图这个输出。

### 验证方法

- `git --version` 能输出版本号
- 若提示"不是内部或外部命令" → 关掉窗口重开；仍不行说明安装时 PATH 没选对，重新装一次

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| `git` 不是命令 | 命令行窗口是装之前开的 | 关掉重开 |
| `git` 不是命令（重开后仍报） | 安装时没选从命令行可用 | 重新安装并选默认 PATH 项 |
| `git --version` 输出乱码 | 编码问题 | 一般不影响使用，截图即可 |

---

<a id="p00-a3"></a>
## A3 · 配置用户名与邮箱

**任务**：执行 `git config --global user.name "你的名字"` 和 `git config --global user.email "你的邮箱"`，然后用 `git config --global --list` 把这两项配置截图。

### 知识点

**为什么必须配**：每次提交都会记录作者信息。不配的话 Git 会用默认值（可能是奇怪的主机名），将来老师看提交记录不知道是谁提交的。

**`--global` 的含义**：对**当前电脑上的所有仓库**生效。不加的话只对当前仓库生效（换个目录又要重配）。

### 操作步骤

```powershell
git config --global user.name "张三"
git config --global user.email "zhangsan@example.com"
```

> 引号里换成你自己的。邮箱建议用你注册 GitHub 的那个邮箱。

验证：

```powershell
git config --global --list
```

**预期输出**（能看到你刚配的两项）：

```
user.name=张三
user.email=zhangsan@example.com
...
```

截图。

### 验证方法

- `git config --global --list` 里能看到你自己的那两行
- 名字和邮箱确实是你自己的（不是示例里的"张三"）

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 输出里找不到 user.name | 输成了 `--list`（少了 `--global`） | 加上 `--global` 再看 |
| 中文名显示乱码 | 终端编码 | 换成拼音或英文名（推荐） |
| 配错了想改 | — | 重新执行同样的命令覆盖即可 |

---

<a id="p00-a4"></a>
## A4 · 建本地仓库

**任务**：新建一个文件夹，在里面放一个 `hello.txt`，用命令把该文件夹变成 Git 仓库（`git init`），并截图 `git status` 的输出。

### 知识点

**`git init` 做了什么**：在文件夹里创建一个隐藏的 `.git` 子目录，所有版本历史都存在那里。文件夹本身**内容没变**，只是"变成了一个仓库"。

**⚠️ 必须在正确的目录执行**。如果你在家目录（`C:\Users\你的名字`）执行 `git init`，会把整个用户目录都变成仓库——非常难清理。

### 操作步骤

1. 建一个练习文件夹，比如 `D:\git-practice`
2. **先确认自己在哪**：

```powershell
pwd
```

   输出应该是 `D:\git-practice`（如果不是，先 `cd D:\git-practice`）
3. 在文件夹里新建 `hello.txt`，随便写一行内容
4. 初始化仓库：

```powershell
git init
```

   预期输出：

```
Initialized empty Git repository in D:/git-practice/.git/
```

5. 查看状态：

```powershell
git status
```

   预期输出（关键三行）：

```
On branch master

No commits yet

Untracked files:
        hello.txt

nothing added to commit but untracked files present
```

6. 截图。

**读懂这段输出**：
- `No commits yet` —— 还没有任何提交
- `Untracked files` —— `hello.txt` Git 看到了，但还没被纳入版本管理
- 这正是"工作区"里的文件，还没 add

### 验证方法

- `git status` 输出里有 `Untracked files: hello.txt`
- 用资源管理器开启"显示隐藏文件"，能看到 `.git` 文件夹
- **确认你的 `.git` 在练习文件夹里，不在用户目录**

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 在错的目录 init 了 | 没先 `pwd` 确认 | 删掉误建的 `.git` 文件夹（只删 `.git`，不删文件） |
| `git init` 报权限错误 | 目录无写权限 | 换到 D 盘用户目录下 |
| 看不到 `.git` | 资源管理器没开隐藏文件 | 查看 → 显示 → 隐藏的项目 |

---

<a id="p00-a5"></a>
## A5 · 第一次提交

**任务**：把 `hello.txt` 提交到仓库：先 `git add`，再 `git commit`（写一条有意义的中文提交信息），然后用 `git log` 截图看到这次提交。

### 知识点

**两步走的原因**：

- `git add` —— 挑选"这次要提交哪些改动"（放进暂存区）
- `git commit` —— 把这些改动打包成一个版本，**并写一句为什么**

**为什么要分两步**：你可以一次只提交部分文件。比如同时改了 A、B 两个文件，但只想先提交 A。

**提交信息怎么写**：一句话说清"**这次改了什么**"。
- ✅ `新增 hello.txt 并写入第一行内容`
- ❌ `aaa`、`更新`、`提交`

### 操作步骤

```powershell
git add hello.txt
git status
```

`git status` 应该变成：

```
Changes to be committed:
        new file:   hello.txt
```

（文件从"未跟踪"变成了"待提交"）

提交：

```powershell
git commit -m "新增 hello.txt 并写入初始内容"
```

预期输出：

```
[master (root-commit) a1b2c3d] 新增 hello.txt 并写入初始内容
 1 file changed, 1 insertion(+)
 create mode 100644 hello.txt
```

看历史：

```powershell
git log
```

预期输出（能看到提交哈希、作者、时间、提交信息）：

```
commit a1b2c3d... (HEAD -> master)
Author: 张三 <zhangsan@example.com>
Date:   ...

    新增 hello.txt 并写入初始内容
```

截图。

### 验证方法

- `git log` 里能看到这次提交，作者是你自己
- 提交信息是有意义的中文（不是"aaa"）
- `git status` 现在显示 `nothing to commit, working tree clean`

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 报 `nothing to commit` | 忘了先 `git add` | 先 add |
| `git log` 后卡住出不来 | 进入了分页器 | 按 `q` 退出 |
| commit 弹出一个编辑器 | 忘了写 `-m "信息"` | 保存退出（或者补上 `-m`） |
| 提交信息是乱码 | 编码问题 | 用英文提交信息，或配置终端为 UTF-8 |

---

<a id="p00-a6"></a>
## A6 · 观察"已修改未提交"

**任务**：修改 `hello.txt` 的内容，再用 `git status` 观察"已修改但未提交"的状态并截图；然后提交第二次。

### 知识点

**三种状态要分清**（这就是 `git status` 的价值）：

| 状态 | 含义 | 能否 commit |
|---|---|---|
| Untracked | Git 没见过这个文件 | 不能（要先 add） |
| Modified（未暂存） | 文件改了，但没 add | 不能（要先 add） |
| Staged（已暂存） | 已 add，等 commit | 能 |

### 操作步骤

1. 打开 `hello.txt`，把内容改写一下并保存
2. 查看状态：

```powershell
git status
```

预期输出：

```
Changes not staged for commit:
        modified:   hello.txt
```

（**注意是 `not staged`** —— 改了但没 add）

3. 截图这个"已修改未暂存"的状态
4. 再走一遍提交流程：

```powershell
git add hello.txt
git commit -m "修改 hello.txt 内容"
git log --oneline
```

`git log --oneline` 会看到**两条**提交：

```
b2c3d4e 修改 hello.txt 内容
a1b2c3d 新增 hello.txt 并写入初始内容
```

### 验证方法

- 观察到过 `Changes not staged for commit` 这个状态
- `git log --oneline` 有 2 条提交记录
- 能说清 `not staged` 和 `to be committed` 的区别

---

<a id="p00-a7"></a>
## A7 · 注册并建远程仓库

**任务**：注册一个 GitHub 账号（用常用邮箱），在网站上新建一个**私有**仓库，仓库名自定。

### 知识点

**本地仓库 vs 远程仓库**：现在你的仓库只在自己电脑上。远程仓库（GitHub 上那个）是**备份 + 分享**的地方——也是老师看作业的入口。

**私有 vs 公开**：

| | 谁能看 | 什么时候用 |
|---|---|---|
| Private（私有） | 只有你自己（和被你邀请的人） | 学习作业、未完成的代码 |
| Public（公开） | 所有人 | 开源项目 |

**任务要求建私有**——因为后面是你的作业，不需要给别人看（要给老师看时，在设置里加协作者即可）。

### 操作步骤

1. 打开 `https://github.com/signup`，用常用邮箱注册（需要邮箱验证码）
2. 登录后，点右上角 **`+`** → **`New repository`**
3. 填写：

   | 字段 | 填什么 |
   |---|---|
   | `Repository name` | 自定，如 `my-stm32-learning`（**不要有空格**，用 `-` 连接） |
   | `Description` | 可选，如"我的单片机学习仓库" |
   | `Public / Private` | 选 **Private** |
   | `Initialize this repository with...` | **全都不勾**（因为我们本地已有仓库，勾了会冲突） |

4. 点 **`Create repository`**
5. 创建后页面会显示一段"…or push an existing repository from the command line"的提示——**先别关，A8 要用到那个 URL**

### 验证方法

- 仓库创建成功，页面显示 `Private` 标记
- 页面上给你的仓库地址形如 `https://github.com/<你的用户名>/my-stm32-learning.git`
- **没有**勾选 README/.gitignore/License（否则后面 push 会冲突）

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 注册收不到验证码 | 邮箱进了垃圾箱 | 查垃圾邮件；或换个邮箱 |
| 仓库名报错 | 含空格或特殊字符 | 只用字母数字和 `-` |
| 后面 push 报 rejected | 建仓库时勾了 README | 先 `git pull --allow-unrelated-histories` 再 push |

---

<a id="p00-a8"></a>
## A8 · 连接远程并推送

**任务**：把这个本地仓库与 GitHub 上的远程仓库连起来，并完成第一次 `push`；推送后在**浏览器打开仓库网页**，确认能看到 `hello.txt` 与两条提交记录，并截图。在这个仓库里写一份 README，把你第 1 条的说明放进去。

### 知识点

**连接两步**：

```
git remote add origin <仓库地址>   ← 记下"远程仓库在哪"，给它起名 origin
git push -u origin master          ← 把本地提交推上去，并记住这个对应关系
```

> **分支名注意**：新版 Git 默认分支可能叫 `main`。用 `git branch` 看你的分支名，把下面命令里的 `master` 换成你实际的（`master` 或 `main`）。

**`-u` 的作用**：记住"本地 master ↔ 远程 origin/master"，以后直接 `git push` 就行。

### 操作步骤

1. 关联远程仓库（地址用 A7 页面上的）：

```powershell
git remote add origin https://github.com/你的用户名/my-stm32-learning.git
git remote -v          # 确认：应显示 fetch 和 push 两行同样地址
```

2. 推送：

```powershell
git push -u origin master
```

> 如果报 `src refspec master does not match any`，你的分支名是 `main`，把 `master` 换成 `main`。

**第一次推送会弹出登录窗口**（关于这个窗口，见下方"常见问题"——这是新手最容易卡住的地方）。

3. 推送成功的输出形如：

```
Enumerating objects: 6, done.
...
To https://github.com/你的用户名/my-stm32-learning.git
 * [new branch]      master -> master
Branch 'master' set up to track 'remote branch master' from 'origin'.
```

4. **打开浏览器**，刷新你的仓库页面，确认能看到：
   - `hello.txt` 文件
   - **两条**提交记录
5. 在仓库里新增 `README.md`（可用记事本创建），把 A1 写的那 100 字说明放进去，然后提交并推送：

```powershell
git add README.md
git commit -m "添加 README，说明 Git 与网盘的区别"
git push
```

6. 截图仓库网页（能看到文件列表和提交历史）。

### 验证方法

- 浏览器里能看到 `hello.txt` 与 `README.md`
- 提交历史里有 ≥2 条记录
- `git remote -v` 显示的地址正确

### 常见问题

**⭐ 反复弹窗要账号密码，输了也不对**

这是新手第一大坑，三个可能原因：

| 原因 | 说明 | 怎么办 |
|---|---|---|
| **密码认证已停用** | GitHub 早就不让用登录密码做 Git 操作了 | 弹出窗口里**用户名填 GitHub 用户名，密码填 Personal Access Token**（不是登录密码） |
| **装了加速镜像** | 本机配了把 `github.com` 改写到其他网站的规则，而镜像站只读、不接受推送 | 检查 `git config --global --get-regexp url.` 有没有输出；有的话说明有改写规则，需要改用官方地址或调整配置 |
| **错误凭据被缓存** | 之前输错过一次，系统记住了错的那份 | 在 Windows「凭据管理器」→「Windows 凭据」里删掉 github.com 相关条目，重新推送 |

**怎么生成 Personal Access Token**：GitHub 网页 → 头像 → `Settings` → 最下面 `Developer settings` → `Personal access tokens` → `Tokens (classic)` → `Generate new token` → 勾选 **`repo`** → 生成后**立刻复制**（只显示一次），把它当密码用。

| 其他现象 | 原因 | 怎么办 |
|---|---|---|
| `remote origin already exists` | 重复 add 了 | `git remote set-url origin <正确地址>` |
| `rejected / non-fast-forward` | 远程有你本地没有的提交 | 先 `git pull --allow-unrelated-histories` 再 push |
| `src refspec master does not match` | 分支名是 main 不是 master | 把命令里的 master 换成 main |
| 推送后网页看不到新提交 | 只 commit 没 push | commit 是存本地，push 才上传 |

---

<a id="p00-a9"></a>
## A9 · 写 .gitignore

**任务**：在仓库里新建 `.gitignore` 文件，把 `.o`、`.hex`、`Debug/`、`Objects/` 这几类编译产物写进去（后面做单片机项目时马上就会用到），提交并推送。

### 知识点

**为什么需要 `.gitignore`**：单片机项目编译后会产生大量**中间文件**（`.o`、`.hex`、`Debug/`、`Objects/`）：

- 它们**每次编译都变**，Git 会认为"一直有改动"，很烦
- 它们**能从源码重新生成**，没必要存
- 它们**体积大**，几百个文件会把仓库塞爆

`.gitignore` 就是告诉 Git："这些文件**不要**纳入版本管理。"

**判断标准**：**能不能从已有文件重新生成？** 能 → 忽略。

### 操作步骤

1. 在仓库根目录新建文件 `.gitignore`（**注意前面有个点**，记事本保存时文件名写 `".gitignore"` 带引号，避免变成 `.gitignore.txt`）
2. 写入以下内容：

```gitignore
# 编译产物
*.o
*.d
*.hex
*.bin
*.elf
*.map

# Keil / 编译输出目录
Debug/
Release/
Objects/
Listings/
MDK-ARM/*/

# 编辑器临时文件
*.bak
*.tmp
Thumbs.db
Desktop.ini
```

3. 提交推送：

```powershell
git add .gitignore
git commit -m "添加 .gitignore，忽略编译产物"
git push
```

4. 刷新网页，确认 `.gitignore` 已上传

### 验证方法

- 网页上能看到 `.gitignore`（且不是 `.gitignore.txt`）
- 文件内容包含 `.o`、`.hex`、`Debug/`、`Objects/` 四类
- 能解释"为什么要忽略这些文件"（能重新生成 + 每次都变）

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 文件名叫 `.gitignore.txt` | 记事本自动加了 .txt | 保存时文件名写 `".gitignore"`（带引号） |
| 已经提交过的产物仍在仓库里 | `.gitignore` 只管"未跟踪"的文件 | 需要 `git rm --cached <文件>` 把它们从仓库移除后再提交 |
| 规则不生效 | 路径写错 | 用 `git check-ignore -v <文件>` 检查规则是否命中 |

---

<a id="p00-b1"></a>
## B1 · 分支与合并

**任务**：建立分支：新建分支 `feature/edit-hello`，在分支上修改 `hello.txt` 并提交，然后把该分支合并回主分支，截图 `git log --graph --oneline` 的结果。

### 知识点

**分支是什么**：一条独立的"改动线"。在分支上试验新想法，**不影响主分支**。试好了再合并回去；试砸了直接删掉分支，主分支毫发无损。

```
master:    A ── B ────────────── M     ← M 是合并提交
                  \             /
feature:           C ── D ──────       ← 在分支上的改动
```

### 操作步骤

```powershell
# 1. 看当前分支
git branch

# 2. 新建并切到分支
git checkout -b feature/edit-hello

# 3. 在分支上改 hello.txt，然后提交
git add hello.txt
git commit -m "在分支上修改 hello.txt"

# 4. 切回主分支（master 或 main）
git checkout master

# 5. 合并分支
git merge feature/edit-hello

# 6. 看图
git log --graph --oneline
```

预期输出：

```
*   f5e6d7c (HEAD -> master) Merge branch 'feature/edit-hello'
|\
| * e4d5c6b (feature/edit-hello) 在分支上修改 hello.txt
|/
* b2c3d4e 修改 hello.txt 内容
* a1b2c3d 新增 hello.txt 并写入初始内容
```

截图。

### 验证方法

- `git log --graph --oneline` 能看到分支与合并的结构（有 `|\` 和 `|/`）
- 主分支上包含了分支的改动
- 能讲清"分支上的改动是怎么进到主分支的"

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| `checkout` 报有未提交改动 | 切分支前工作区不干净 | 先提交或撤销改动 |
| 合并后内容不对 | 两个分支改了同一处（冲突） | 打开冲突文件，按提示保留想要的内容，再 `git add` + `git commit` |
| 不知道主分支叫啥 | 新旧 Git 不同 | `git branch` 看带 `*` 的那个 |

---

<a id="p00-b2"></a>
## B2 · 改坏了怎么救回来

**任务**：做一次"改坏了再救回来"的练习：修改 `hello.txt` 后**先不提交**，用命令把这次修改撤销掉，确认文件回到修改前的样子并截图。

### 知识点

**这是 Git 最实用的功能**。撤销分几种情况，命令不同：

| 你想撤销什么 | 命令 |
|---|---|
| 工作区改动，还没 add | `git restore <文件>` |
| 已经 add 了，想取消暂存 | `git restore --staged <文件>` |
| 已经 commit 了，想回退 | `git revert <提交>`（安全）或 `git reset`（改写历史，慎用） |

**关键理解**：**只要 commit 过，任何东西都能找回来**。所以养成"每完成一点就 commit"的习惯，你就有了无数个存档点。

### 操作步骤

```powershell
# 1. 先确认当前是干净的
git status

# 2. 故意改坏 hello.txt（比如删掉几行乱写）

# 3. 确认 Git 看到改动了
git status          # 应显示 modified: hello.txt

# 4. 撤销这次改动（还没 add 的情况）
git restore hello.txt

# 5. 确认文件回到修改前
git status          # 应显示 nothing to commit, working tree clean
```

打开 `hello.txt` 检查内容确实恢复了。截图。

### 验证方法

- 撤销后 `git status` 显示干净
- 文件内容确认回到修改前
- 能说出"已 add"和"未 add"两种情况下的撤销命令区别

### 常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| `git restore` 没效果 | 已经 add 过了 | 先 `git restore --staged hello.txt` 再 `git restore hello.txt` |
| 报 `unknown option` | Git 版本太老 | `git checkout -- hello.txt` 是旧写法，也能用 |
| 撤销完发现想要那些改动 | — | 如果之前 commit 过，能从历史里找回；**没 commit 过的就真的没了** |

---

<a id="p00-b3"></a>
## B3 · clone 与下载压缩包的区别

**任务**：把远程仓库上的内容用 `clone` 重新拉取到另一个文件夹，打开确认文件与提交历史完整，说明 `clone` 与"下载压缩包"的区别。

### 知识点

| | `git clone` | 下载 ZIP |
|---|---|---|
| 得到什么 | 所有文件 **+ 完整历史** | **只有当前这一版**文件 |
| 能提交吗 | **能**，是个完整仓库 | **不能**，改了传不回去 |
| 能看历史吗 | 能 | 不能 |
| 用途 | 真正开始工作 | 只想看看代码 |

**一句话**：ZIP 是"快照"，clone 是"把整个仓库搬回来"。

### 操作步骤

```powershell
# 换到另一个目录，比如 D:\
cd D:\

git clone https://github.com/你的用户名/my-stm32-learning.git clone-test

cd clone-test
git log --oneline          # 应该能看到你的全部提交历史
```

**对照实验**：在网页上点绿色 `Code` 按钮 → `Download ZIP`，解压后：

- 里面有文件，但**没有 `.git` 目录**
- 在解压出来的目录里执行 `git log` 会报 `not a git repository`

把两者对比写进 README。

### 验证方法

- clone 出来的目录里 `git log --oneline` 能看到完整历史
- ZIP 解压出来的目录里 `git log` 报错（证明没有历史）
- 能讲清区别，并说出"为什么协作要用 clone 而不是 ZIP"

---

<a id="p00-b4"></a>
## B4 · 在网页上看提交历史

**任务**：在 GitHub 网页上查看自己仓库的提交历史，点开某一次提交，讲清"这次提交改了哪几行"；把截图放进 README。

### 知识点

**GitHub 网页的三种"看历史"入口**：

| 位置 | 看什么 |
|---|---|
| 文件列表上方的 **commits** 链接（或时钟图标） | 全部提交列表 |
| 任意文件页面的 **History** 按钮 | 该文件的修改历史 |
| 提交列表里点某次提交 | 这次提交的**具体差异**（红色是删除、绿色是新增） |

**"Blame" 视图**：文件页面上的 `Blame` 按钮，能看到**每一行是哪次提交、谁、什么时候加的**。排查"这行代码为什么这么写"时极有用。

### 操作步骤

1. 打开你的仓库网页，点文件列表上方的提交数（如 `3 commits`）
2. 点开其中一条提交，观察页面：
   - 红色 `-` 行 = 删除的
   - 绿色 `+` 行 = 新增的
3. 截图，并在旁边用文字说明"这次提交改了哪几行、为什么改"
4. 把截图与说明加进 README 并推送

### 验证方法

- 能指着某次提交说清"红色这几行被换成了绿色这几行"
- 能在文件历史里看到同一个文件被多次修改
- README 里放了截图 + 说明

---

<a id="p00-b5"></a>
## B5 · 写一份能看懂的 README

**任务**：写一份让不看代码的人也能看懂的 README：包含本仓库是做什么的、目录里有什么、怎么运行或查看。提交并推送。

### 知识点

**README 是第一印象**。别人（包括将来的你）打开仓库先看它。合格标准是：**不看代码也知道这个仓库是干什么的、里面有什么。**

**必备三块**：

1. **这个仓库是做什么的**（一两句话）
2. **目录结构**（每个文件/文件夹干什么）
3. **怎么用**（怎么看/怎么运行/怎么跑起来）

### 操作步骤

照这个模板写：

```markdown
# my-stm32-learning

我的单片机学习仓库，记录学习过程中的练习代码与笔记。

## 目录结构

| 路径 | 说明 |
|---|---|
| `hello.txt` | 练习文件，用来练手 Git 操作 |
| `.gitignore` | 忽略编译产物的规则 |

## 关于 Git 与网盘的区别

（把 A1 那 100 字放这里）

## 怎么查看

浏览器打开本仓库即可阅读所有文件；想改代码请用 `git clone` 拉到本地。

## 提交历史

（放 B4 的截图）
```

提交推送：

```powershell
git add README.md
git commit -m "完善 README：说明仓库用途与目录结构"
git push
```

### 验证方法

- 请一位同学看你的 README，问他"这个仓库是干什么的"——他能答出来
- 三块内容齐全（用途 / 目录 / 怎么用）
- 在 GitHub 仓库首页能直接看到（README.md 会自动渲染在最前面）

---

<a id="p00-b6"></a>
## B6 · 提交信息的重要性

**任务**：做一次"提交回顾"实验：故意写一条无意义的提交信息（如"aaa"），然后说明为什么这样的信息在半年后对你自己也没用，并把正确的写法写进 README。

### 知识点

**提交信息是写给未来的自己看的**。半年后你只会看这些信息来回忆"当时改了什么"。写"aaa"等于什么都没记。

**好提交信息的三条规则**：

1. **说清"改了什么"**，而不是"我改了"
   - ✅ `修复按键抖动导致重复触发`
   - ❌ `改了一下`
2. **一条提交只做一件事**（别把三个改动塞进一次提交）
3. **必要的话补一句"为什么"**

**反例对照**：

| 差 | 好在哪 |
|---|---|
| `aaa` | 半年后完全不知道改了什么 |
| `更新` | 更新了啥？ |
| `修改代码` | 修改了哪个文件的什么逻辑？ |
| ✅ `修复 P03 去抖漏判：确认态未清导致长按重复上报` | 一看就知道改了什么、为什么 |

### 操作步骤

1. 故意做一次烂提交：

```powershell
# 随便改点东西
git add .
git commit -m "aaa"

# 看历史
git log --oneline
```

2. 观察 `git log --oneline` 里那条 `aaa`——思考：半年后看到它，你知道改了什么吗？
3. 把上面"三条规则 + 反例对照表"写进 README
4. 提交这次 README 更新（**这次用一条规范的提交信息**）：

```powershell
git add README.md
git commit -m "补充 README：提交信息规范与反例对照"
git push
```

> 那两条 `aaa` 的提交不用删（删了反而麻烦）。让它留着，当作一个反面教材。

### 验证方法

- 能讲清"为什么 `aaa` 这样的信息对自己也是无效的"
- README 里有明确的提交信息规范
- 后续的提交信息都符合规范（评审会翻你的提交历史）

---

## 附：日常最常用的 8 条命令

| 命令 | 用途 |
|---|---|
| `git status` | 现在什么状态（**最常用，出问题先敲它**） |
| `git add <文件>` | 把改动放进暂存区 |
| `git commit -m "说明"` | 存成一个版本 |
| `git log --oneline` | 看历史（简洁） |
| `git push` | 推到 GitHub |
| `git pull` | 把远程的改动拉下来 |
| `git restore <文件>` | 撤销工作区改动 |
| `git checkout -b <分支名>` | 新建并切到分支 |

---

## 下一步

- 回到任务卡 `tasks/P00-Git与GitHub入门.md` 逐条勾选
- 交付物：一个自己的练习仓库（含 `hello.txt`、`.gitignore`、README）
- **把仓库地址发给带教人**——开课后第一周会检查一次
- 这个仓库后面会一直用：P01 开始，把单片机工程提交到它的 `projects/` 目录下
