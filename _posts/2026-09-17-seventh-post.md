---
layout: post
title: "POST"
tags: web
date: 2026-09-17
---
<h1>解题核心：脱离页面直接构造请求</h1>

# 细节原理
- 参数必须在请求体里，不能写在 URL 后
- 地址栏改 ?what=flag 是没用的
- 必须借助页面表单、JS 控制台、Python 脚本或工具来构造 POST 请求
- 用浏览器控制台发 POST 请求
  
# 输入代码
- fetch(当前网址, ...) — 浏览器去请求当前页面
- method:'POST' — 用 POST 方式发请求
- headers: {...} — 告诉服务器这是个普通表单
- body:'what=flag' — 请求体里塞 what=flag
- r.text() — 把返回的网页内容读成文本
- document.body.innerText = t — 把读到的文本显示在页面上
