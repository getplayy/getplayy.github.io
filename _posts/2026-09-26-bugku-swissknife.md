---
layout: post
title: "【Bugku CTF】瑞士军刀 Writeup"
date: 2026-09-26 21:20:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：PWN
- **分值**：10 分（入门）
- **考察点**：nc（netcat）远程连接 + 裸 TCP 交互 + Windows 工具链的换行符陷阱

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

题目名「瑞士军刀」——指的就是 nc（netcat），网络工具界的瑞士军刀。开启环境后给出一个地址：`nc IP 端口`。

## 解题过程

### 0x01 原理：nc 是什么

nc 能和远程服务器的端口建立**裸 TCP 连接**——连上后，**键盘输入直接发给服务器，服务器输出直接回显给你**。这题服务器开了个远程 shell，用 nc 连上去敲命令读 flag 即可。

与 WEB 题的本质区别：WEB 题隔着浏览器和 HTTP 协议；这题你和服务器之间**只有一个裸 TCP 连接**——像直接坐在服务器终端前。

### 0x02 题目环境给出的三个坑

| 坑 | 说明 |
|----|------|
| IP:端口 vs IP 空格端口 | 题目写 `nc 160.202.254.160:10338`，但 nc 命令里**用空格**：`nc 160.202.254.160 10338` |
| 连上后黑屏无回显 | **正常现象**，服务器不发欢迎语，直接盲敲命令即可 |
| Ctrl+C 会断开 | nc 会话里 Ctrl+C = 断连，得重连 |

### 0x03 Windows 上的连环坑（本文重点）

理论上一条 `nc IP 端口` 就完事，但 **Windows 没有自带 nc**，于是开启了一段工具链排坑之旅：

**尝试 1**：cmd 里输 `nc` → `'nc' 不是内部或外部命令` ❌

**尝试 2**：PowerShell 一行流模拟 nc → 连上了，但报错：

```text
cat: 'flag\r': No such file or directory
```

**原因**：PowerShell 的 `WriteLine()` 发送的是 Windows 换行 `\r\n`，Linux 服务器只认 `\n`——**多出的 `\r` 被当成了文件名的一部分**，服务器找的是名为 `flag\r` 的文件，当然不存在。这次失败反而证明了连接是通的、命令在执行。

**尝试 3**：改用 `Write()` 手动控制换行符 → 命令在 cmd→powershell 的**双层引号转义**中损坏，粘贴进 cmd 后回车无反应 ❌

**最终方案**：直接执行——

```bash
printf 'ls\ncat flag\n' | nc 160.202.254.160 10338
```

一次发送 `ls` + `cat flag` 两条命令（`\n` 是正确的 Linux 换行），输出：

```text
bin
dev
flag
lib
lib32
lib64
pwn1
flag{***}
```

`ls` 看到根目录有 `flag` 文件，`cat flag` 读出 flag，提交 +10 分 ✅

### 0x04 Windows 用户的备选工具链

| 方案 | 命令/操作 | 评价 |
|------|----------|------|
| 原生 nc | `nc IP 端口`（需安装） | 最标准，Kali 自带 |
| PowerShell TcpClient | 见下方 | 零安装，但注意换行符 |
| 双击 .bat 工具 | 固化连接逻辑，输入 IP 端口即可 | 对新手最友好 |

PowerShell 正确姿势（核心是 `$w.Write()` + 手动 `` `n ``，**不是** `WriteLine()`）：

```powershell
$c = New-Object System.Net.Sockets.TcpClient('IP', 端口)
$s = $c.GetStream()
$w = New-Object System.IO.StreamWriter($s); $w.AutoFlush = $true
$w.Write("ls`ncat flag`n")
Start-Sleep 2
$b = New-Object byte[] 8192
$n = $s.Read($b, 0, 8192)
[System.Text.Encoding]::ASCII.GetString($b, 0, $n)
```

> PowerShell 里换行符写法是反引号+n（`` `n ``），不是 `\n`——这是和 Linux 脚本最大的语法差异之一。

## 原理分析

### nc：网络瑞士军刀

nc 支持端口扫描、文件传输、开监听、当客户端——渗透测试必备。本题只用到了它最基础的**客户端模式**：

```
你 ──键盘──> nc ──TCP──> 服务器:10338
你 <─屏幕── nc <─TCP── 服务器:10308
```

### 换行符：一个字符引发的血案

三套系统的换行约定不同：

| 系统 | 换行符 | 十六进制 |
|------|--------|---------|
| Linux/macOS | `\n`（LF） | `0A` |
| Windows | `\r\n`（CRLF） | `0D 0A` |
| 老 Mac | `\r`（CR） | `0D` |

当 Windows 程序把 `\r\n` 发给 Linux shell，shell 按 `\n` 分割命令，**`\r` 留在了命令末尾**——`cat flag` 变成了 `cat flag\r`，文件名多了个不可见字符，必然找不到文件。**跨平台协议交互的经典陷阱**，真实场景中 HTTP 头、FTP 命令、邮件协议都靠严格约定换行符工作。

### 为什么这题归 PWN 类

远程 shell 交互是 PWN 的**最外层入口**——真正的大门（缓冲区溢出、格式化字符串、ROP 链）在它后面。这题算"PWN 前置技能送分题"，考点是**熟悉远程连接工具和裸 TCP 交互**，不涉及漏洞利用。

## 复盘

- 解题路径：理解 nc → Windows 三连坑（无 nc → `\r` 污染 → 引号转义损坏）→ 正确换行符一次通过，全程约 1 小时（大部分时间耗在 Windows 工具链上）
- **本次最大的收获不是 flag，是三个工程教训**：
  1. **报错信息是最好的老师**：`cat: 'flag\r'` 这句报错直接暴露了换行符问题——**读报错，别只看"失败"两个字**
  2. **长命令别走多层解释器**：cmd → powershell 的双层引号转义极脆弱，复杂命令写成 .ps1/.bat 文件执行，不 inline
  3. **工具要固化**：一次调通的命令做成脚本存起来（.bat 双击即用），下次同类题直接跑
- **环境时效意识**：题目环境一般几十分钟过期，连接失败先想"是不是过期了"，重开环境端口会变
- 知识链更新：WEB（交互层）→ MISC（文件层）→ Crypto（编码层）→ **PWN（系统/网络层，第 13 题开启新板块）**
- 同类题识别：题目给 `nc IP 端口` 形式的提示 → 连上直接 `ls` + `cat flag`，无回显别慌，换行符要纯净

## 参考

- [netcat 维基百科](https://zh.wikipedia.org/wiki/Netcat)
- [PowerShell 换行符 `` `n `` 与 `` `r `` 官方文档](https://learn.microsoft.com/zh-cn/powershell/module/microsoft.powershell.core/about/about_special_characters)
- [CRLF 与 LF：换行符的历史](https://zh.wikipedia.org/wiki/%E6%8F%9B%E8%A1%8C)
