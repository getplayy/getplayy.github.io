---
layout: post
title: "【Bugku CTF】滑稽 Writeup"
date: 2026-09-19 12:00:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：WEB
- **分值**：10 分（入门）
- **考察点**：前端源码信息泄露（HTML 注释）

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

听说聪明的人都能找到答案。

## 解题过程

### 0x01 侦察页面

打开题目环境，满屏都是动态的滑稽笑脸，除此之外没有任何输入框、按钮等交互点，动画由两个 JS 文件驱动。

遇到这类"纯观赏"页面，先默念 CTF 前端第一定律：**页面上看不到的东西，都在源码里。**

### 0x02 查看源代码

`F12` 打开开发者工具，在 Elements 面板里 `Ctrl + F` 搜索关键词 `flag`，直接命中——flag 就写在 HTML 注释中：

```html
<!-- flag{***} -->
```

带上下文的源码大致如下：

```html
<!doctype html>
<html>
<head>...</head>
<body>
    <!-- flag{***} -->
    <script type="text/javascript" src="js/ThreeCanvas.js"></script>
    <script type="text/javascript" src="js/Snow.js"></script>
</body>
</html>
```

*（此处配一张 F12 定位到注释的截图）*

### 0x03 拿分

复制注释中的 flag，回平台提交，+10 分。

## 原理分析

### 渲染层 vs 源码层

浏览器拿到一份 HTML 文档后会经历 **解析 → 构建 DOM 树 → 渲染** 三个阶段。`<!-- -->` 是 HTML 的注释语法，解析阶段被跳过、渲染阶段不呈现，所以肉眼看不到；但它作为 HTTP 响应体的一部分，已经原封不动地发到了客户端——**"没显示"不等于"不存在"**。

### 信息泄露

本题真正考察的安全概念是**前端敏感信息泄露**。真实场景中，开发者经常把测试账号、API Key、内部接口、后台路径写进注释，上线时忘记清理。对攻击者而言，查看源码是零成本动作；对防御者而言，**凡是下发给浏览器的内容，都必须视为公开信息**。

对应防御手段：

- 生产构建时自动剥离注释（Webpack / Vite 等构建工具默认支持）
- 密钥类信息永远只存在服务端
- 代码审计时将 HTML 注释与 TODO / FIXME 一并检查

## 复盘

- 解题路径：无交互页面 → F12 → 搜 `flag` → 注释命中，全程不到 1 分钟
- **Elements ≠ 查看源代码（`Ctrl + U`）**：Elements 显示浏览器解析后的 DOM，可被 JS 动态修改；`Ctrl + U` 显示服务器返回的原始响应。遇到"JS 执行后抹掉关键信息"的题，`Ctrl + U` 是兜底手段
- 延伸习惯：拿到任何页面先 `Ctrl + U` 扫一遍注释、`meta`、隐藏 `input`，再看 Network 面板
- 同类题识别：页面什么都看不到 / 提示"答案就在眼前" → 大概率是源码、请求头、响应头三处信息泄露之一

## 参考

- [MDN：HTML 基础](https://developer.mozilla.org/zh-CN/docs/Learn/Getting_started_with_the_web/HTML_basics)
- [OWASP WSTG：HTML 注释中的敏感信息](https://owasp.org/www-project-web-security-testing-guide/)

---

## 附：writeup 中书写 HTML 标签与注释的方法

写 writeup 经常要展示网页源码。直接在 Markdown 正文里敲尖括号标签，发布后会被当成真实 HTML 处理：标签可能被浏览器吞掉，注释会直接消失。三种正确姿势：

**1. 行内代码** —— 行文中偶尔提到某个标签，用反引号包裹：`` `<div>` `` 渲染为 `<div>`。

**2. 围栏代码块** —— 展示源码片段（本文用法），内部内容原样输出，无需任何转义：

```html
<!-- flag{***} -->
<div>hello</div>
```

**3. HTML 实体转义** —— 需要在正文裸文本中显示尖括号时，将 `<` 写作 `&lt;`、`>` 写作 `&gt;`，例如 `&lt;!--flag--&gt;` 渲染为 `<!--flag-->`。

> 进阶：若要在代码块里展示"三反引号代码块"本身的写法，外层围栏需用四个反引号，否则会提前闭合。
