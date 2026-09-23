---
layout: post
title: "【Bugku CTF】POST Writeup"
date: 2026-09-19 12:00:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：WEB
- **分值**：10 分（入门）
- **考察点**：POST 传参 / 脱离页面直接构造请求

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

页面直接给出一段 PHP 源码：

```php
$what = $_POST['what'];
echo $what;
if ($what == 'flag')
    echo 'flag{***}';
```

要求用 POST 方式传入 `what=flag`，即在请求体里塞这个参数。

## 解题过程

### 0x01 审计源码

这题和「GET」代码几乎一样，**唯一区别是 `$_GET` 换成了 `$_POST`**。也就是说：

- 参数必须在请求体里，不能写在 URL 后
- 地址栏改 `?what=flag` 是没用的
- 必须借助页面表单、JS 控制台、Python 脚本或工具来构造 POST 请求

### 0x02 用浏览器控制台发 POST 请求

F12 → 切到「控制台」标签（中文 Edge/Chrome 在顶部第二或第三个标签）→ 第一次粘贴会弹黄色安全警告，按提示手动输入「**允许粘贴**」回车 → 之后粘贴下面这行代码 → 回车：

```javascript
fetch(location.href, {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'what=flag'
}).then(r => r.text()).then(t => document.body.innerText = t)
```

**这段代码的逐段意思**：

- `fetch(当前网址, ...)` — 浏览器去请求当前页面
- `method:'POST'` — 用 POST 方式发请求
- `headers: {...}` — 告诉服务器这是个普通表单
- `body:'what=flag'` — 请求体里塞 `what=flag`
- `r.text()` — 把返回的网页内容读成文本
- `document.body.innerText = t` — 把读到的文本显示在页面上



### 0x03 拿分

页面上会直接渲染出 `flag{***}`，复制回 Bugku 提交框粘贴，提交成功 +10 分。

### 备选姿势（不写也行，了解即可）

**姿势二：Python 脚本**（先装 requests：`pip install requests`）

```python
import requests
r = requests.post('http://你的题目地址/', data={'what': 'flag'})
print(r.text)
```

**姿势三：Hackbar 插件**（Firefox 附加组件，搜索安装）→ 加载 URL → 勾选 Enable Post data → 填 `what=flag` → Execute。

## 原理分析

### POST 与 GET 的区别

| | GET | POST |
|---|---|---|
| 参数位置 | URL 中，`?key=value` | 请求体（body）中 |
| 地址栏能直接发 | ✅ | ❌ |
| 肉眼可见 | 地址栏直接看到 | 需抓包或工具 |
| 典型用途 | 查询、分享链接、分页 | 登录、提交数据、文件上传 |

PHP 用超全局变量 `$_GET` / `$_POST` 分别接收两种来源。

### 页面只是构造请求的途径之一

本题的关键认知和「计算器」「GET」一脉相承：**服务器只认"收到一个 POST 请求，body 是 what=flag"，不管这个请求从哪来**。本题页面没有表单，所以需要借助控制台 fetch、Python 脚本等"绕过页面"直接跟服务器对话——这正是 Web 安全黄金法则的应用：

> 客户端不可信。服务端只看请求本身，请求的构造方式（页面表单 vs 脚本 vs curl）服务端一概不知。

### 浏览器粘贴安全提示

控制台粘贴代码时出现的黄色警告"Don't paste code that you don't understand"是浏览器的安全机制——防止社会工程学攻击（如钓鱼页面诱导受害者粘贴"修复代码"来窃取 Cookie / 跳转危险网站）。手动键入「允许粘贴」是显式确认：用户已知风险并主动授权。这个细节本身也是 CTF 安全概念的一部分（社会工程学 vs 用户教育）。

## 复盘

- 解题路径：读代码 → 发现要 POST → 浏览器控制台 fetch → 页面回显 flag → 提交，全程 2 分钟
- **关键卡点**：`Ctrl + V` 第一次粘贴会被安全警告拦截，必须手动输入「允许粘贴」回车才能解锁
- 知识链：滑稽（**看**源码）→ 计算器（**改**前端）→ alert（**解**编码）→ GET（**构造 URL**）→ POST（**构造请求体**）——五道题连成 Web 前端攻防的完整视角
- 延伸：真实场景中登录、发帖、上传文件都是 POST；遇到没有表单的 POST 接口（API 调用），控制台 fetch 就是最快的调试/测试手段
- 同类题识别：题目给源码要求 POST，但页面没有表单 → 必须用工具/脚本直接构造请求体

## 参考

- [MDN：HTTP 请求方法](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods)
- [PHP 手册：$_POST](https://www.php.net/manual/zh/reserved.variables.post.php)
- [MDN：fetch()](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API/Using_Fetch_API)
