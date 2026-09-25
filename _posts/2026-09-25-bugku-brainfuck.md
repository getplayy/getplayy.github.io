---
layout: post
title: "【Bugku CTF】[+-<>] Writeup"
date: 2026-09-25 23:39:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：Crypto
- **分值**：10 分（入门）
- **考察点**：Brainfuck 语言识别与解释 + 与 Ook! 的对照

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

题目名 `[+-<>]`——本身就是 Brainfuck 的指令字符子集。密文只由八种字符组成：`+ - < > . , [ ]`，形如：

```text
++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]（示例，以自己页面为准）
```

## 解题过程

### 0x01 识别：八字符指纹

题目名直接剧透——密文只含 Brainfuck 的 8 种指令字符，这就是 **Brainfuck 语言**，上一题「ok」里 Ook! 的"本体"。

识别指纹：**只含 `+-<>.,[]` 八种字符的纯符号流 → 必是 Brainfuck**。

### 0x02 splitbrain 解码：换按钮就行

上一题刚用过 [splitbrain.org/services/ook](https://www.splitbrain.org/services/ook)，这题同站换一个按钮：

1. 密文整段粘进大文本框
2. 点 **`Brainfuck to Text`**（不是上题的 `Ook! to Text`）
3. 文本框里直接出明文 ✅

```text
flag{***}
```

*（此处配一张 splitbrain 解出明文的截图）*

### 0x03 拿分

复制提交，+10 分。

### 备选：CyberChef（注意参数和 Ook 题相反）

CyberChef → 搜索 `Brainfuck` → 拖入 `To Brainfuck` → **参数保持默认 `Standard`** → 密文粘 Input → Output 出明文。

> 对比记忆：Ook! 题**必须把参数从 `Standard` 改成 `Ook`**；本题是标准 Brainfuck，**参数保持 `Standard` 不动**。两题唯一的配置差异就在这里。

## 原理分析

### Brainfuck：8 条指令的图灵机

Brainfuck（1993，Urban Müller 设计）模拟一台极简计算机——**一条无限长的纸带**（每格存一个字节，初始为 0）+ **一个指针**：

| 指令 | 含义 |
|------|------|
| `>` | 指针右移一格 |
| `<` | 指针左移一格 |
| `+` | 当前格的值 +1 |
| `-` | 当前格的值 -1 |
| `.` | **输出**当前格的 ASCII 字符 |
| `,` | 读入一个字符存入当前格 |
| `[` | 若当前格为 0，跳到匹配的 `]` |
| `]` | 若当前格非 0，跳回匹配的 `[` |

想输出字母 `A`（ASCII 65）？65 个 `+` 再一个 `.`：

```text
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++.
```

**就这 8 条指令，却图灵完备**——理论上可以计算任何可计算问题（代价是写起来痛苦万分）。

### 图灵完备：值得记住的概念

**图灵完备（Turing-complete）** = 能模拟通用图灵机 = 能算任何可计算的东西。判断核心：支持条件分支 + 无限存储（或等价物）。Brainfuck 用 `[ ]` 循环 + 无限纸带达标。

有趣的事实：PowerPoint、万智牌（Magic: The Gathering）规则、x86 的 MOV 单指令——都被证明图灵完备。**"能计算"的门槛远比想象低**，这是理解计算本质的一扇窗。

### 与 Ook! 的关系：语言与语法糖

上一题的 Ook! 与 Brainfuck **指令一一对应**（`Ook. Ook?`=`>`、`Ook. Ook.`=`+`……），只是换了写法。两题连着做，直观体会到：

- **语言的核心是语义**（这 8 个操作），**写法只是皮**（`>` vs `Ook. Ook?`）
- 解释器处理 Ook! 的第一步就是 token 替换成 Brainfuck，再执行——所谓"语法糖"就是这样一层翻译

## 复盘

- 解题路径：八字符指纹识别 → splitbrain 换按钮 `Brainfuck to Text` → 明文，全程 1 分钟
- **工具复用**：上一题踩过的坑（CyberChef 加载慢、参数找不到）这次直接绕开——splitbrain 一次解决两题，**同一类问题积累一个顺手的工具**是效率关键
- **指纹库再添一条**：`+-<>.,[]` 八字符 → Brainfuck；`Ook. Ook? Ook!` → Ook!；以后遇到 `[]()!+` → JSFuck（Web 进阶会遇到）
- **入门 12 题至此收官**：WEB 7 题（滑稽/计算器/alert/GET/POST/头等舱/你必须让他停下——交互层）+ MISC 1 题（单纯的图片——文件层）+ Crypto 4 题（栅栏/摩斯/Ook/Brainfuck——编码层），三个视角全部点亮 🎉

## 参考

- [Esolang Wiki：Brainfuck](https://esolangs.org/wiki/Brainfuck)
- [Wikipedia：图灵完备性](https://zh.wikipedia.org/wiki/%E5%9B%BE%E7%81%B5%E5%AE%8C%E5%A4%87%E6%80%A7)
- [splitbrain Ook!/Brainfuck 解释器](https://www.splitbrain.org/services/ook)
