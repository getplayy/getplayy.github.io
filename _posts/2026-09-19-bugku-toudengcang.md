---
layout: post
title: "【Bugku CTF】头等舱 Writeup"
date: 2026-09-19 12:00:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：WEB
- **分值**：15 分（入门偏进阶）
- **考察点**：HTTP 响应头信息泄露

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

头等舱。

## 题目分析

**题名是最大线索**——"头等舱"谐音"标头等（舱）"，直指 **HTTP Header**。提示区里也有人反馈"bp 抓包之后在 Target 里面直接看"、"在网络点开第一个去看响应头中的 data"，确认解题路径。

## 解题过程

### 解法一：F12 网络面板（推荐新手）

1. **F12** 打开 DevTools → 切到 **「网络」** 标签
2. **按 F5 刷新**（Network 只记录它打开之后的请求）
3. 点开列表中**第一条请求**（一般就是当前文档）
4. 切到 **「标头」** 或 **「Headers」** 面板
5. 找到 **「响应标头」**（Response Headers）部分
6. 在字段里找到 `flag` 开头的那一行，形式如：

```http
flag: flag{xxxxx}
```

7. 复制整段提交平台 ✅

### 解法二：Burp Suite

1. 浏览器配置 Burp 代理（默认 `127.0.0.1:8080`，安装 Burp CA 证书到浏览器）
2. 浏览器打开题目地址
3. 切到 Burp **Target → 站点地图** → 展开题目域名
4. 点开具体请求 → 在 **Response** 区的 **Headers** 中找 flag

> Burp 提示里有人推荐"Target 里面直接看"，是新手最省事的路径。

### 解法三：curl（写脚本时最实用）

终端执行：

```bash
curl -I http://题目地址
```

`-I` 表示只看响应头。响应头一次性打印出来，在里面找 flag；想更快就用 grep：

```bash
curl -sI http://题目地址 | grep -i flag
```

## 原理分析

### HTTP 交互的两块信息

浏览器和服务器之间一次 HTTP 请求，传输的内容有两块：

| 部分 | 谁发出的 | 内容 | 是否渲染到页面 |
|------|---------|------|--------------|
| 响应头（Response Headers） | 服务器 | 描述本次响应的元信息（格式、缓存、服务器身份、…） | ❌ 不渲染 |
| 响应体（Response Body） | 服务器 | 实际的页面内容（HTML / 图片 / JSON…） | ✅ 渲染 |

**响应头**承载了“这次返回是什么类型、怎么缓存、跨域放不放行”等配套说明，肉眼在浏览器里**看不到**，必须通过 DevTools / Burp / curl 才能看到。

### 信息泄露家族

这题与「滑稽」「alert」同属**信息泄露**大类，三者泄露的藏身处各有不同：

| 题 | 藏身处 | 发现方式 |
|-----|--------|----------|
| 滑稽 | HTML 注释 | Elements 面板 |
| alert | HTML 注释 + 编码 | Elements + CyberChef |
| **头等舱** | **HTTP 响应头** | **F12 网络 / Burp / curl** |

**安全启示**：凡是“页面上看不到”的地方——注释、head/header、隐藏 input、Cookie、Response Body 里被注释掉的字段——都是信息泄露的高发区。渗透测试的标准动作是：拿到任意 URL 后第一件事就是把页面源码 + 响应头 + Cookie + Meta 标签全部过一遍。

### HTTP 头常见泄露与防御

| 泄露字段 | 风险 | 防御 |
|---------|------|------|
| `flag: ...` | 题目特意埋的答案 | 生产环境移除调试标识 |
| `Server: nginx/x.x.x` | 暴露服务器版本 → 已知漏洞利用 | 隐藏或改写 Server 头 |
| `X-Powered-By: PHP/...` | 暴露运行时版本 | 配置中关闭 |
| `Set-Cookie: ...` | 会话劫持凭据 | HttpOnly、Secure 标志 |
| `Access-Control-Allow-Origin: *` | CORS 配置过宽 | 按需精确放行 |

**黄金法则**：**响应头里的任何信息都应视为对用户公开的**。

## 复盘

- 解题路径：识别题名暗示 → F12 网络面板看响应头 → 提交，全程不到 2 分钟
- **第一次正式用上「网络/Network」面板**，这是 Web 渗透的核心工具之一
- **第一次正式接触 Burp Suite**，即使这题没用上，后续「你必须让他停下」「Web6 速度要快」都重度依赖 Burp，建议尽早装好（Community 版免费）
- 同类题识别：页面"啥都没有"+ 题名/描述有"头"、"标头"、"header"、"cookie" 等字眼 → 第一反应查响应头
- 写作时进时建议：把三种解法都写上——F12 给“最小依赖”、Burp 给“专业工具”、curl 给“脚本能力”——readers 容易举一反三

## 参考

- [MDN：HTTP Headers](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers)
- [MDN：HTTP 响应头字段参考](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers#%E5%93%8D%E5%BA%94%E5%9C%BA%E6%99%AF)
- [Burp Suite 下载](https://portswigger.net/burp/communitydownload)