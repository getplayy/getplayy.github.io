---
layout: post
title: "【Bugku CTF】linux & Linux2 Writeup"
date: 2026-09-28 21:55:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：MISC
- **分值**：linux 15 分 + Linux2 25 分（入门）
- **考察点**：Linux 命令行（tar / file / strings / grep）+ 二进制文件字符串提取

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

**linux**：附件 `1.tar.gz`，提示 `key{}`。
**Linux2**：附件 `brave.zip`，解压得到无后缀的 `brave` 文件（约 20 MB），提示 key 的格式是 `KEY{}`。

两道姊妹题，考点同源：在一堆二进制数据里把字符串捞出来。

## 解题过程

### 0x01 linux：解压与识别

```bash
$ file 1.tar.gz
1.tar.gz: gzip compressed data, from Unix

$ tar -zxvf 1.tar.gz
test/
test/flag
```

解压出 `test` 文件夹，里面是一个**无后缀**的 `flag` 文件。按惯例先 `file` 一下：

```bash
$ file test/flag
test/flag: Linux rev 1.0 ext3 filesystem data
```

原来是块 **ext3 文件系统镜像**——题目叫 "linux" 的由来。

### 0x02 打捞字符串

直接 `cat` 是一大屏乱码，拉到**最后几行**能看到明文 key。更优雅的做法：

```bash
$ strings test/flag | grep -i key
key{***}
```

`strings` 提取文件里所有可打印字符串，`grep` 负责过滤——一条管道秒杀肉眼翻找。

*（此处配一张终端 strings+grep 的回显截图）*

### 0x03 Linux2：当心干扰项

解压 `brave.zip` 得到无后缀的 `brave`。手顺 `foremost` 做文件分离，**确实分出一张图片，图上明晃晃写着 `flag{...}`——但提交是错的**。

回头看提示：格式是 `KEY{}`，图片上那个连格式都对不上，早就该起疑。正解仍是老一套：

```bash
$ strings -a brave | grep "KEY{"
KEY{***}
```

### 0x04 拿分

两个 flag 分别提交，15 + 25 分到手。

## 原理分析

### 字符串打捞：MISC 的基本功

CTF 里大量题目归结为一句话：flag 藏在文件某个角落，把它找出来。解法分三层：

1. **肉眼层**：`cat` / `tail` / 十六进制编辑器翻看（费眼，兜底用）；
2. **工具层**：`strings | grep`（本题正解）；
3. **自动化层**：`grep -rE "key\{|flag\{" .` 递归扫目录树（文件多时）。

### grep 遇二进制会"装哑巴"

不给参数时 grep 对二进制文件只报 `Binary file xxx matches`，不显示内容——必须加 `-a` 才按文本处理。这是新手最常卡住的一步。

### 题目提示即钥匙

两题的提示都只给了**格式**（`key{}` / `KEY{}`），它同时是搜索关键字、结果校验标准、干扰项过滤器——Linux2 那张假 flag 图片，正是被格式校验拦下的。

## 复盘

- 解题路径：file 识别 → 解压 → strings+grep 打捞，正解全程不到 1 分钟；Linux2 多绕了一步 foremost 的坑
- **Windows 做题实况**：无后缀文件双击弹"选择应用"→ 选 VS Code → 又弹"二进制或不受支持的编码"警告 → 点"仍要打开" → `Ctrl+F` 搜 `key{`。三连坑走完才体会到：Linux 下一条 `strings | grep` 的事，这就是题目叫 linux 的原因。以后取证类题目切 WSL
- **VS Code 小技巧**：二进制警告是保护机制（防手滑改坏文件），点"仍要打开"只搜不改即可；也可以装 Hex Editor 扩展看十六进制视图
- **文件大别慌**：20 MB 的 brave 不用"看完"，strings+grep 秒级定位——工具思维，不是人肉思维
- 知识链回顾：滑稽（看源码）→ 计算器（改前端）→ alert（解编码）→ GET/POST（构造请求）→ 散乱的密文（换位置）→ 富强民主（换字符集）→ **linux/Linux2（命令行捞字符串）**——从 Web/编码正式踏入取证方向
- 下一题预告：隐写类（改图片宽高 / LSB），`strings` 这招马上还要再用

## 参考

- [Bugku 题目页 - linux](https://ctf.bugku.com/challenges/detail/id/15.html)
- [Bugku 题目页 - Linux2](https://ctf.bugku.com/challenges/detail/id/19.html)
- [Linux 命令大全：strings](https://man.linuxreviews.org/man1/strings.1.html)
