---
layout: post
title: "【Bugku CTF】telnet Writeup"
date: 2026-09-29 20:55:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：MISC
- **分值**：入门
- **考察点**：流量分析入门——pcap 文件 + Wireshark 追踪 TCP 流

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

附件解压得到 `networking.pcap`——一个抓包文件。题目名 telnet。

## 解题过程

### 0x01 认识 pcap：一段被录下来的网络

`.pcap` = packet capture，抓包工具录下的**真实网络流量**：谁连了谁、发了什么、回了什么，全按时间顺序存在里面。分析它需要抓包界的"播放器"——[Wireshark](https://www.wireshark.org/)（官网下载，一路 Next，比装 IDA 省心一百倍）。

### 0x02 过滤 + 追踪流：两步出 flag

用 Wireshark 打开 `networking.pcap`：

1. 顶部过滤栏输入 **`telnet`** 回车——只显示 telnet 协议的包（题目名就是过滤条件）
2. 随便右键一个包 → **Follow（追踪流）→ TCP Stream（TCP 流）**

弹出的窗口里是这条 telnet 会话的完整内容：登录过程 + 明文数据，flag 就躺在里面：flag{***}

*（此处配一张 Wireshark 追踪 TCP 流的截图）*

### 0x03 不装工具的偷懒解

pcap 说到底是个二进制文件，明文 flag 就藏在里面——**linux 篇的 `strings | grep` 第二次应验**：

```bash
strings networking.pcap | grep flag
```

或者 Windows 老三样：VS Code 打开 → "仍要打开"二进制警告 → `Ctrl+F` 搜 `flag{`（Linux2 的三连坑复刻）。

## 原理分析

### telnet 为什么"裸奔"

telnet 是 1969 年代的远程登录协议，设计时网络安全还不存在：**账号、密码、命令全部明文传输**。同一网段任何人抓个包，你的登录凭证一览无余。这就是 SSH 出现的原因——

| | telnet | SSH |
|---|---|---|
| 端口 | 23 | 22 |
| 传输 | **明文** | 全程加密 |
| 抓包可见性 | 密码直接可读 | 只有密文 |
| 现状 | 已淘汰 | 远程登录标准 |

这道题附件本质就是"一次被抓了包的 telnet 登录"——**明文传输 = 对抓包者零保密**，这是本题想让你记住的一句话。

### TCP 流重组：135 个包拼成一段话

细看过滤结果会发现：telnet 登录时**一个字母一个包**（客户端敲一个字、服务器回显一个字），一次登录要几十上百个包。逐包看等于一个字一个字地拼——而 **Follow TCP Stream** 做的就是"流重组"：把同一 TCP 连接的双向数据按序拼成完整会话。这是流量分析的第一个必会操作。

### Burp 和 Wireshark 的分工

做题时自然冒出的疑问：都是抓包，为什么 Burp 不行？

| | Burp Suite | Wireshark |
|---|---|---|
| 本质 | 中间人**代理** | 抓包**分析器** |
| 工作时机 | 流量"正在发生"时拦截 | 实时抓 **或** 回放 pcap 录像 |
| 协议范围 | 只懂 HTTP/HTTPS | 全部（telnet/DNS/TCP/……） |
| 能开 pcap | ❌ | ✅ 主业 |

记法：**Burp 是安检门，只查现在过门的 HTTP 车；Wireshark 是监控室，什么车都能看还能回放录像**。手里一盘录像带（pcap），只有监控室能放。

## 复盘

- 解题路径：认出 pcap → Wireshark 打开 → telnet 过滤 → 追踪 TCP 流 → 复制 flag，全程 1 分钟
- **strings 应验清单**：linux（第 1 次）→ telnet（第 2 次）——"二进制文件里捞明文"已经是肌肉记忆了
- **工具地图再添一城**：Burp（HTTP 实时拦截）+ Wireshark（全协议 + pcap 回放）= 抓包双雄，各管一段
- **偷懒与长线的平衡**：VS Code 搜索能解这题，但下一道流量题（协议细节分析、pcap 修复、USB 流量还原）必须真 Wireshark 上场——这次先记下它的位置
- 知识链回顾：linux/Linux2（静态文件取证）→ **telnet（网络流量取证）**——取证分支从"死文件"扩展到"活流量"；与"你必须让他停下"（Burp 拦 HTTP）隔着 Web/杂项两个方向遥相呼应
- 下一题预告：流量分析还有进阶玩法（协议统计、文件提取、USB 键盘流量），Wireshark 装上就是为它们准备的

## 参考

- [Wireshark 官网](https://www.wireshark.org/)
- [RFC 854：Telnet Protocol Specification](https://www.rfc-editor.org/rfc/rfc854)
- [Wireshark 官方文档：Following TCP Streams](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvFollowTCPSection.html)
