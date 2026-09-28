---
layout: post
title: "Git 踩坑实录：一次博客部署引发的六连坑"
date: 2026-09-28 22:45:00 +0800
categories: git
---

- **环境**：Windows + PowerShell + VS Code + GitHub Pages（Jekyll）
- **分类**：工具链踩坑记录
- **考察点**：Git 工作流 + 报错排错方法论

> 个人学习记录。写 CTF writeup 博客的第一天，git 上连踩六个坑，全部现场记录——每一个都是新手必经之路。

## 背景

博客用 GitHub Pages + Jekyll 搭建，流程是：往 `_posts` 里丢 Markdown → `git push` → Pages 自动构建发布。听起来三步，实际踩了六坑，从下午陪到深夜。按时间线复盘。

## 踩坑过程

### 0x01 坑一：明明 push 了，文章没上去

**现象**：

```text
Untracked files:
  _posts/2026-09-28-bugku-linux-Linux2.md
nothing added to commit but untracked files present
Everything up-to-date
```

push 显示"成功"，网站却没更新。

**原因**：git 的三步流水线被我走成了只剩最后一步。打个比方：`git add` 是**装箱**，`git commit` 是**贴单封箱**，`git push` 是**发货**——我没装箱没封箱直接喊发货，快递员看仓库里没有新包裹，只能说"没什么可发的"（Everything up-to-date）。

**修复**：

```powershell
git add _posts/2026-09-28-bugku-linux-Linux2.md
git commit -m "新增 Bugku linux & Linux2 writeup"
git push origin main
```

**坑中坑**：`git commit`（不带 `-m`）会弹出 COMMIT_EDITMSG 编辑器界面，第一行空着直接关标签页 = 放弃提交。正确姿势：第 1 行写提交信息 → 关掉标签页，提交瞬间完成。以 `#` 开头的行是说明注释，不会被提交。

*（此处配一张 git status 显示 Untracked files 的截图）*

### 0x02 坑二：`git: 'pull--rebase' is not a git command`

**现象**：敲 `git pull--rebase`，报"不是 git 命令"。

**原因**：`pull` 和 `--rebase` 之间**少了个空格**。子命令和选项必须分开写——纯粹的手滑，但配上英文报错还是懵了两秒。

**修复**：`git pull --rebase origin main`，拉取并变基。

### 0x03 坑三：`fatal: not a git repository`（犯了两次）

**现象**：连跑四条命令全部报 `fatal: not a git repository (or any of the parent directories): .git`。

**原因**：**站错目录了**。提示符明明白白写着 `PS D:\gitblog>`，而仓库在子目录 `D:\gitblog\getplayy.github.io` 里。git 命令只对"当前所在目录"生效——在外层普通文件夹执行，它向上找不到 `.git`，全部拒绝。同一个坑掉进去两次，间隔半小时，值得记录。

**修复**：

```powershell
cd D:\gitblog\getplayy.github.io
git status    # 验证：能看到 "On branch main" 才说明站对了
```

**知识点**：PowerShell 提示符里的路径 = 当前位置。**每次开终端第一件事：看提示符在不在仓库里，不在先 `cd`**。

*（此处配一张提示符路径对比截图）*

### 0x04 坑四：本地和远端分叉（diverged）

**现象**：commit 时 COMMIT_EDITMSG 头部提示：

```text
# Your branch and 'origin/main' have diverged,
# and have 1 and 1 different commits each, respectively.
```

**原因**：本地有 1 个提交没推上去，远端也有 1 个本地没有的提交（在 GitHub 网页端操作会产生），两条线各走各的。此时直接 push 会被拒绝。

**修复**：

```powershell
git pull --rebase origin main    # 把本地提交"接到"远端最新提交之后
git push origin main
```

**如果 rebase 报冲突**（两边动了同一个文件）：VS Code 里打开冲突文件，保留需要的版本、删掉 `<<<<<<<` / `=======` / `>>>>>>>` 标记行，然后 `git add 文件名` → `git rebase --continue`。

**保险丝**：rebase 途中任何时候觉得乱了，`git rebase --abort` 一键回到操作前状态，什么都不丢。

### 0x05 坑五：502 Bad Gateway

**现象**：push 时网关报错 502。

**排查思路**：502 属于 5xx——**服务端/链路的锅，不是我的操作错了**。先查 [githubstatus.com](https://www.githubstatus.com/)：全绿，Git Operations 100% 正常 → GitHub 没挂，是**我到 GitHub 之间的链路抽风**（代理节点不稳最常见）。

**对策**（按顺序）：

1. 等 1~2 分钟直接重试——502 多是阵发性的；
2. 连试几次不行：浏览器开 github.com 对照——浏览器也打不开 = 换节点/热点；浏览器能开但 git 不行 = 浏览器走了代理而 git 没走，给 git 配上：

```powershell
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
# 端口以自己的代理工具为准；取消：git config --global --unset http.proxy
```

*（此处配一张 502 报错 + GitHub Status 全绿的对照截图）*

### 0x06 坑六：VS Code 的报错弹窗是个"话痨终结者"

**现象**：同步时弹错误框，只显示 `From https://github.com/...`——这明明是 pull 的**正常输出第一行**，真正报错被截断了，误导我找了半天。

**原因**：VS Code 的 git 弹窗永远只显示命令输出的第一行。

**对策**：**判断 git 状态一律以终端为准**——信息全、可滚动、可复制。弹窗只能当"有事发生了"的提醒铃。

## 原理分析

### Git 三区模型：所有坑的总根源

```text
工作区          暂存区           本地仓库          远端仓库
(编辑文件) --add--> (装箱) --commit--> (封箱) --push--> (发货)
     <------------ pull / fetch + merge/rebase ------------
```

- 坑一 = 跳过 add/commit 直接 push；
- 坑四 = 本地仓库和远端仓库各自前进后需要 pull 对齐；
- 其余的坑都在"执行层"（目录、拼写、网络、工具）——模型清楚了，报错一眼定位是哪一环。

### 以后 push 前的固定节奏

```powershell
cd 仓库目录            # 0. 先站对地方
git status            # 1. 万能探针：红/绿/干净 一目了然
git add -A            # 2. 装箱
git commit -m "信息"   # 3. 封箱（信息写清楚改了什么）
git pull --rebase origin main   # 4. 先捋直分叉
git push origin main  # 5. 发货
```

把第 4 步变成 push 前的固定动作，就永远不会再遇到 diverged。

### 排错方法论

1. **先看提示符**——确认自己在哪，命令才有意义；
2. **git status 探路**——任何 git 操作前先跑它，红（改动）、绿（暂存）、"nothing to commit"（干净）三种状态一看便知；
3. **分清 4xx 和 5xx**——4xx 是自己的操作问题，5xx 是服务端/链路问题，先查 [status 页](https://www.githubstatus.com/)再折腾自己；
4. **以终端全文为准**——GUI 弹窗只当门铃。

## 复盘

六坑速查表，存一张贴手边：

| # | 坑 | 一句话根因 | 一句话修复 |
|---|---|---|---|
| 1 | push 了没更新 | 跳过 add/commit | 三步走全：add → commit → push |
| 2 | not a git command | 子命令和选项少空格 | `pull --rebase` 分开写 |
| 3 | not a git repository ×2 | 站在仓库外层目录 | `cd` 进仓库，`git status` 验证 |
| 4 | diverged | 本地远端各有新提交 | `pull --rebase`；冲突改完 `--continue` |
| 5 | 502 | 代理/链路抽风 | 重试 → 对照浏览器 → 配代理 |
| 6 | 弹窗误导 | VS Code 只显示第一行 | 一切以终端输出为准 |

- 最深教训：**同一个坑掉两次（坑三）**——不是不会，是没有形成"开终端先看路径"的习惯。习惯比知识值钱
- 提交信息要写"做了什么"（新增xxx/修复xxx），今天的教训都记在提交历史里了
- 知识链回顾：这是博客基建篇——CTF writeup（GET → 散乱的密文 → 富强民主 → linux/Linux2）负责产出内容，git 负责把内容送出去，两条线今天正式汇合
- 下一站：流程跑通后，考虑给仓库加 GitHub Actions 自动部署，或者研究下 Jekyll 主题美化

## 参考

- [Pro Git 中文版（官方免费书）](https://git-scm.com/book/zh/v2)
- [GitHub Status 状态页](https://www.githubstatus.com/)
- [VS Code 中的 Git 版本控制](https://code.visualstudio.com/docs/sourcecontrol/overview)
