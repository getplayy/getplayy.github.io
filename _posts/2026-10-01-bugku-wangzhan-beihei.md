---
layout: post
title: "【Bugku CTF】网站被黑 Writeup"
date: 2026-10-01 00:40:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：WEB
- **分值**：入门
- **考察点**：目录扫描找 WebShell 后门 + **Burp Intruder 爆破首秀**

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

打开解题链接是一个"被黑"的黑页——深色背景、炫酷特效、源码翻遍没线索。题目名"网站被黑"：黑客入侵后留下的页面，那么入侵的**后门**（WebShell）是不是也还留在服务器上？

页面底部一句警告："**不是自己的马不要乱骑！**"——"马"是黑话，指木马/WebShell。出题人已经把答案预告了。

## 解题过程

### 0x01 目录扫描：后门无处藏身

黑页明面上没东西，就得**猜路径**。目录扫描工具（御剑/dirsearch/dirb）的原理就是拿一本"常见路径字典"挨个访问，200 就是存在、404 就是没有。

本题沙箱快速探测（等价于迷你目录扫描）：

```bash
for p in shell.php webshell/shell.php admin.php login.php ...; do
  curl -s -o /dev/null -w "%{http_code} /$p" http://目标/$p
done
```

结果**一发命中**：

```text
200 /shell.php     ← 后门就在根目录
404 /webshell/shell.php
404 /admin.php
...
```

### 0x02 WebShell 登录页

打开 `/shell.php`——一个简陋的密码框，POST 提交，参数名 `pass`。这就是黑客留的后门：**一个能远程执行命令的 PHP 脚本，用密码防止被别人利用**。

### 0x03 Burp Intruder 爆破：密码 hack

Intruder 四步流程（Burp 三件套的最后一件，首秀）：

1. **Positions**：`Clear §` 清空自动标记 → 只选中密码值 → `Add §` → `pass=§test123§`
2. **Payloads**：Simple list 装字典（hack/admin/123456/password/root……）
3. **Start attack**：逐个替换重发
4. **Length 排序**：错误的响应都长一个样（"密码错误"短响应），**正确那条的长度与众不同**——密码 **`hack`**

浏览器输 `hack` 登录 → flag{***}

## 原理分析

### WebShell：入侵者的"备用钥匙"

WebShell = 以网页文件形式（PHP/JSP/ASPX）存在的后门，上传到服务器后通过浏览器访问就能**远程执行命令**（浏览文件、提权、挂马）。攻击链通常：找上传漏洞 → 传 WebShell → 拿 WebShell 密码自己留着 → 随时回来。

**防御视角**：部署后要扫描 WebShell 特征（D 盾、河马）；本题反过来——把"找后门+破解后门"当成攻防演练。

### 爆破的本质：字典 × 自动重发

| 概念 | 说明 |
|---|---|
| Intruder | 把请求里标记的变量逐个换成字典词重发 |
| Payload | 字典（攻击载荷）——Simple list 只是最低级的一种 |
| 长度判别 | 成功响应的内容（flag 页）和失败响应（错误提示）长度不同 |

爆破能不能成，**字典质量决定一切**——密码 `hack` 是题目名的直译，属于"出题人字典"必收词。真实世界的字典攻击靠的是泄露密码库（rockyou.txt 千万级）+ 目标信息定制的社工字典。

### Burp 三件套集齐

| 模块 | 登场题 | 本题角色 |
|---|---|---|
| Proxy + HTTP history | 本地管理员 | 找到 POST 请求 ✅ |
| Repeater | 本地管理员 | （备用）改包重发 |
| **Intruder** | **本题** | 字典爆破密码 ✅ |

从"看请求"到"改请求"再到"批量发请求"——Burp 主力模块一网打尽。

## 复盘

- 解题路径：黑页无线索 → 目录扫描（沙箱 curl 循环）命中 /shell.php → Intruder 字典爆破 → 密码 hack → flag
- **Intruder 首秀要点**：Clear § 清自动标记（Burp 默认标记一堆位置，不清会跑成笛卡尔积）→ 只标密码值 → Length 排序找异常——这套肌肉记忆以后每道爆破题都用
- **沙箱半程 + 本地半程**的协作模式成型：扫描类脏活沙箱跑（快），工具技能类（Intruder）自己练（涨经验）
- 新概念入库：WebShell（网页后门）、"马"的黑话、字典攻击、长度判别法
- 知识链回顾：变量1（审代码）→ 本地管理员（改请求）→ source（泄露考古）→ **网站被黑（找后门+爆破）**——Web 四连，攻防视角完整闭环：找漏洞 → 利用 → 爆破 → 拿权限
- 下一题预告：评论区同系列还有"输入密码查看flag"（五位纯数字密码爆破，10 万次循环——正好练 Intruder 数字 Payload 或 Python 脚本）、"成绩单"（SQL 注入首秀）

## 参考

- [PortSwigger Web Security Academy：Using Burp Intruder](https://portswigger.net/burp/documentation/desktop/tools/intruder)
- [Wikipedia：Web shell](https://en.wikipedia.org/wiki/Web_shell)
- [OWASP：Password Attack Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
