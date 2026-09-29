---
layout: post
title: "【Bugku CTF】本地管理员 Writeup"
date: 2026-09-29 23:10:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：WEB
- **分值**：入门
- **考察点**：信息收集（源码藏 Base64）+ IP 伪造（X-Forwarded-For）+ **Burp Suite 实战首秀**

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

打开解题链接是一个管理员登录界面，提交任何账号密码都提示：**"IP 禁止访问，请联系本地管理员登录，IP 已被记录"**。题目名：本地管理员。

## 解题过程

### 0x01 信息收集：密码就躺在源码里

`F12` → Elements 面板翻到页面底部（或 `Ctrl+F` 搜 `==`），找到一段 Base64：

```text
dGVzdDEyMw==
```

解码得 `test123`——"藏在页面里的密码"，这密码谁用？当然是管理员。账号顺理成章猜 `admin`。

但直接提交还是被拒——IP 检查这道墙没过。

### 0x02 题目名即提示：让服务器以为你在"本地"

"本地管理员"四个字翻译成技术语言：**只允许 `127.0.0.1`（本机回环地址）访问**。你当然不可能跑到服务器本机去登录，但 HTTP 请求头里有个字段可以"撒谎"——`X-Forwarded-For`。剩下的就是把这个头塞进登录请求里。

### 0x03 Burp 实操：从被垃圾流量淹没到精准改包

**踩坑第一幕：Intercept on 的暴击**

一开始走"实时拦截"路线：开 Intercept → 浏览器提交表单。结果请求还没发，队列先堆了 9 个包——`phingest.portswigger.com`（Burp 自家遥测）、`edge.microsoft.com`（Edge 同步）、`doubleclick.net`（广告追踪）……**一开拦截，所有后台流量全堵在门口**。

**换工作流：HTTP history 事后回看**

关掉 Intercept（队列瞬间清空放行），先在浏览器正常登录一次（失败无所谓），然后去 Proxy → **HTTP history** 页签找录像——列表最底下就是刚才那个 POST：

```http
POST / HTTP/1.1
Host: 160.202.254.160:11512
...

user=admin&pass=test123
```

> 顺带确认了真实参数名是 **`user` / `pass`**（不是想当然的 username/password——改包前先看包，这是教训也是铁律）。

**最后一击：Repeater 改包重放**

右键该请求 → **Send to Repeater** → 在 `Host:` 下面加一行请求头：

```http
X-Forwarded-For: 127.0.0.1
```

点 **Send**——响应不再是登录表单，flag{***} 到手。

## 原理分析

### X-Forwarded-For：本来是代理的"传话"，成了攻击者的"伪造"

`X-Forwarded-For`（注意拼写 **Forwarded**，最容易敲错）的正当用途：客户端经过代理/负载均衡访问后端时，后端看到的来源 IP 是代理的；代理于是在这个头里**记录客户端的真实 IP**，一路传递。

问题在于：**这个头是客户端可以随意填写的**。如果后端拿它判断"你从哪来"而不做任何校验，IP 限制就成了摆设——你说自己是谁，服务器就信你是谁。本题正是这个逻辑缺陷的教科书复刻：服务器信任 `X-Forwarded-For` 判断"是否本地"，攻击者填上 `127.0.0.1` 即可冒充本机。

**真实世界同款风险**：绕过 WAF 封禁、绕过地域限制、伪造投票/签到 IP。防御姿势：只在可信的反向代理链路上信任此头，或干脆用 TCP 层真实来源 IP。

### 一家人：还有哪些头能"报 IP"

| 请求头 | 说明 |
|---|---|
| `X-Forwarded-For` | 事实标准，最常被后端读取 |
| `X-Real-IP` | Nginx 常配的单 IP 记录头 |
| `Client-IP` / `X-Client-IP` | 老写法，偶尔有程序读 |
| `X-Forwarded-Host` | 伪造 Host 用，另一道题的伏笔 |

做题时一个不行就挨个试——后端到底读哪个头，只有试了才知道。

### Burp 三件套分工：这道题把核心工作流全练了一遍

| 模块 | 角色 | 本题用途 |
|---|---|---|
| Proxy + Intercept | 安检门（实时拦截） | 新手坑：垃圾流量全排队 ❌ |
| Proxy + HTTP history | 监控室（事后回放） | 找到登录 POST ✅ |
| Repeater | 改包重放台 | 加 XFF 头反复调试 ✅ |

telnet 篇记下的分工表可以更新了：**Burp 不只是"安检门"，它自带监控室（history）和重放台（Repeater）——实战中后两者才是主力**。实时拦截留给"必须原地拦下修改"的场景（如"你必须让他停下"那题）。

## 复盘

- 解题路径：F12 捞密码（Base64 → test123）→ 题目名翻译成 IP 限制 → Burp 抓 POST → Repeater 加 `X-Forwarded-For: 127.0.0.1` → 登录成功
- **Burp 首秀三连坑实录**：① Intercept on 被垃圾流量淹没（微软遥测+广告商排队）；② 实时拦截不如事后回看——HTTP history 工作流从此定为默认；③ 参数名是 `user/pass` 不是 `username/password`——**改包前先看包**
- **Web 方向第二战**：变量1 是"读代码找漏洞"，本题是"看提示想伪造"——代码审计与逻辑漏洞两条腿走路
- 新姿势入库：`X-Forwarded-For` 伪造 IP（附全家桶变体）、Burp Proxy/History/Repeater 三件套、Base64 特征识别（`==` 结尾）已成本能
- 知识链回顾：linux/Linux2（认文件）→ telnet（抓流量）→ 眼见非实（拆容器）→ 变量1（审代码）→ **本地管理员（改请求）**——telnet 篇那句"Burp 是安检门"正式升级为完整工具链认知
- 下一题预告：同款思路还有得玩——`X-Forwarded-Host` 伪造、HTTP 响应头泄密、Burp Intruder 爆破（本题账号 admin 其实就是 Intruder 一把梭出来的）

## 参考

- [MDN：X-Forwarded-For 请求头](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/X-Forwarded-For)
- [PortSwigger Web Security Academy：Using Burp Repeater](https://portswigger.net/burp/documentation/desktop/tools/repeater)
- [PortSwigger Web Security Academy：HTTP Request Tampering](https://portswigger.net/web-security/request-tampering)
