---
layout: post
title: "【Bugku CTF】easy_nbt Writeup"
date: 2026-09-30 23:10:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：MISC
- **分值**：入门（1 万+ 人解出）
- **考察点**：文件格式识别（gzip 魔数）+ Minecraft NBT 存档结构 + 字符串搜索

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

附件解压后是一个 **Minecraft（我的世界）游戏存档**——一堆文件夹和 `.dat` 文件。题目名 easy_nbt 的 NBT = Named Binary Tag，Minecraft 的数据存储格式，游戏里的物品、方块、玩家数据全用它存。

```text
存档目录/
├── level.dat          ← 世界信息
├── playerdata/        ← 玩家数据（物品栏、坐标……）
│   └── <UUID>.dat     ← 玩家文件，文件名是 UUID
├── region/            ← 地形区块
└── ...
```

## 解题过程

### 0x01 侦察：认识存档结构

解压后几十个文件，不慌——**先看题目名**。NBT → 搜一下 → Minecraft 存档格式。锁定两个高价值目标：`level.dat`（世界信息）和 `playerdata/<UUID>.dat`（玩家物品栏——出题人把 flag 写在玩家拿的书里）。

### 0x02 魔数识别：.dat 的真身是 gzip

《眼见非实》篇的魔数表今日扩容：`.dat` 文件头是 **`1F 8B`**——gzip 压缩的魔数。套娃结构：**`.dat`（gzip）→ 解压 → NBT 二进制 → 里面明文 UTF-8 存着字符串**（书的内容、物品名……）。

### 0x03 解压：Windows 上的翻山之路（本题真正的boss）

Linux 上三步完事：`binwalk -e level.dat` → `cat` 解压产物 → `grep flag`。Windows 上全是坑，实录如下：

| 尝试 | 结果 | 原因 |
|---|---|---|
| `tar -xzf level.dat` | `Unrecognized archive format` | Windows tar 只认 **tar 归档**，不认"gzip 压缩的裸数据" |
| 右键「全部解压缩」 | 菜单里没有 | 该机器 Win11 未启用 gz 原生解压 |
| 装 7-Zip（winget） | `No package found` | 包名敲错（`7zip.zip` ≠ `7zip.7zip`） |
| **.NET GZipStream** | ✅ 成功 | PowerShell 调用系统内置 .NET，零安装 |

最终解法——PowerShell 四行（**Windows 解 gzip 的保底技能**）：

```powershell
$in  = [IO.File]::OpenRead("D:\New World\level.dat")
$out = [IO.File]::Create("D:\nbt_out\level_nbt.dat")
$gz  = New-Object IO.Compression.GZipStream($in, [IO.Compression.CompressionMode]::Decompress)
$gz.CopyTo($out); $gz.Close(); $out.Close()
```

### 0x04 搜索：嘲讽页与真 flag

记事本打开解压产物（乱码正常——二进制里夹着可读文本），`Ctrl+F` 搜 `flag`。

第一个发现的是**嘲讽页**——level.dat 里一本成书的内容：

```json
[{"text":"where is flag?"}]
```

flag 当然不在这（出题人的恶趣味）。真正的 flag 藏在 **`playerdata/<UUID>.dat`** 解压后的玩家物品栏里——玩家手里那本书的正文：flag{***}

## 原理分析

### NBT：Minecraft 的数据格式

NBT 是树状的二进制标签格式：每种数据类型一个编号（字节=1、字符串=8、列表=9……），层层嵌套存下整个世界。**关键特性：字符串以明文 UTF-8 存储**——所以二进制文件里能直接搜到可读文本，和 telnet 篇的 `strings` 思路同源。

### gzip 两种形态：tar 包 vs 裸数据

| 形态 | 文件头 | 解压方式 |
|---|---|---|
| `.tar.gz` | `1F 8B`（外层 gzip） | `tar -xzf` ✅ |
| gzip 压缩的裸数据（NBT 就是） | `1F 8B` | tar ❌ / .NET GZipStream ✅ |

同为 `1F 8B`，内层结构不同待遇就不同——**tar 要求内层是 tar 归档**，NBT 是裸数据所以被拒。这个坑踩明白了，以后见 `1F 8B` 就知道该用什么解。

### 游戏取证：CTF 的一个流派

用游戏存档出题是 MISC 的经典玩法——Minecraft（NBT）、RPG Maker（存档修改）、模拟器（savestate）。通用思路：**存档 = 结构化数据文件 → 格式识别 → 解压/解析 → 搜敏感字符串**。玩家会走"导入游戏翻箱倒柜"的沉浸路线，CTF 选手走"解压搜字符串"的工程路线——本题 1 万人解出，九成走的是后者。

## 复盘

- 解题路径：认出 MC 存档 → 魔数 `1F 8B` 定 gzip → 解压（tar 坑 → .NET GZipStream 通）→ 搜字符串 → 越过嘲讽页 → playerdata 收 flag
- **Windows 工具链排坑实录**：tar 不认裸 gzip、右键菜单藏 7-Zip（Win11"显示更多选项"）、winget 包名要敲准（`7zip.7zip`）——每个坑都是真实世界的通行税，踩过一次就是资产
- **.NET GZipStream 四行**正式入库：Windows 上解 gzip 的零依赖保底方案，与 7-Zip、全部解压缩并列为三件套
- **记事本搜索的坑**：Ctrl+F 只跳第一个匹配——`where is flag?` 挡在前面，真 flag 在后面。**搜到不等于搜完，F3（下一个）要按到没**
- **魔数清单扩容**：`50 4B`（ZIP/docx）之后新增 `1F 8B`（gzip/tar.gz/NBT）——眼见非实的知识直接应验，两题是同一知识点的两块拼图
- 知识链回顾：linux/Linux2（认文件）→ telnet（抓流量）→ 眼见非实（拆容器）→ 变量1（审代码）→ 本地管理员（改请求）→ **easy_nbt（游戏取证）**——MISC 回归，"解压 + 搜字符串"这个最朴素的方法论第 4 次应验
- 下一题预告：NBT 之后，MISC 还有一整个"文件头/隐写/修复"家族等着——魔数表会越来越长

## 参考

- [Minecraft Wiki：NBT 格式](https://zh.minecraft.wiki/w/NBT%E6%A0%BC%E5%BC%8F)
- [RFC 1952：GZIP 文件格式规范](https://www.rfc-editor.org/rfc/rfc1952)
- [Microsoft 文档：GZipStream 类](https://learn.microsoft.com/dotnet/api/system.io.compression.gzipstream)
- [NBTExplorer（图形化 NBT 查看器）](https://github.com/jaquadro/NBTExplorer)
