---
layout: post
title: "【Bugku CTF】变量1 Writeup"
date: 2026-09-29 22:35:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：WEB
- **分值**：入门（2 万+ 人解出）
- **考察点**：PHP 代码审计入门——可变变量 `$$` + 超全局数组 `$GLOBALS`

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

打开解题链接，页面直接把 PHP 源码亮给你，提示语 `flag In the variable !`——flag 在某个变量里。

```php
flag In the variable !
<?php
include "flag1.php";                      // flag 藏在这个文件里
if (isset($_GET['args'])) {
    $args = $_GET['args'];
    if (!preg_match("/^\w+$/", $args)) {  // 输入只允许字母/数字/下划线
        die("args error!");
    }
    eval("var_dump($$args);");            // ★ 破绽：可变变量
}
?>
```

## 解题过程

### 0x01 新题型：源码全给你，自己找漏洞

Crypto/MISC 是"给你密文/文件自己想办法"，Web 代码审计题反过来——**源码明明白白摆着，考你能不能读懂逻辑、找出破绽**。逐行看：

- `include "flag1.php"`：flag 在服务器上的这个文件里，大概率以 `$flag1 = "flag{...}"` 形式定义，被 include 进来后就是当前脚本的一个全局变量
- `preg_match("/^\w+$/", $args)`：输入只能是字母数字下划线——单引号、分号、括号全被堵死，常规注入死路
- `eval("var_dump($$args);")`：**可变变量**——这就是那扇没关的窗户

### 0x02 可变变量 `$$`：变量名也能动态拼

PHP 的 `$$args` 读作"**先取 `$args` 的值，再把这个值当变量名**"：

```php
$hello = "world";
$test  = "hello";
echo $$test;   // $test 的值是 "hello" → $$test 等价于 $hello → 输出 world
```

所以 `?args=xxx` 时，服务器执行的是 `var_dump($xxx)`——**我能让服务器 dump 任意一个全局变量**，只要名字是纯字母数字下划线。flag 变量叫什么？不知道，但 PHP 有个"全都要"的选项——

### 0x03 payload：一锅端

在 URL 后面加参数：

```text
http://目标地址/index1.php?args=GLOBALS
```

页面 dump 出一整坨数组：`_GET`、`_POST`、`_SERVER`……以及混在其中的 **`flag1`**，值就是 flag{***}，复制提交。

## 原理分析

### `$GLOBALS`：全局变量的仓库

`$GLOBALS` 是 PHP 超全局数组，**自动收录当前脚本所有全局变量**（`_GET`、`_POST`、`flag1`……全在里面）。它的存在逻辑很简单：变量名 → 值的映射本身也是一个变量可访问的数据结构。于是：

```php
$args = "GLOBALS";
eval("var_dump($$args);");   // $$args → $GLOBALS → 全局变量一锅端
```

**flag 变量叫什么名字根本不重要**——这是 `GLOBALS` 比猜变量名优雅的地方。

### 正则为什么拦不住

`/^\w+$/` 只放行 `\w`（字母数字下划线）。它是防注入的：单引号、空格、分号都进不来，`var_dump($$args)` 里塞不进第二个语句。但 `GLOBALS` 恰好是 7 个纯字母——**完全合法**。防御者把"注入代码"的门焊死了，却忘了"读取变量"本身就是授权操作。安全漏洞不总是"突破防御"，很多时候是**防御没覆盖的合法路径**。

### eval()：本就不该出现在生产代码里

`eval()` 把字符串当 PHP 代码执行，是官方文档都标注"非常危险"的函数。本题把用户输入拼进 eval 字符串，属于教科书级的反面教材（真实世界同类漏洞通常叫代码注入 / 命令注入）。看到 `eval` + 用户输入拼接，条件反射：**这里有洞**。

### 备用解：猜变量名

`include "flag1.php"` 暴露了文件名，flag 变量多半就叫 `$flag1`——`?args=flag1` 同样能出。但万一变量叫 `$flag`、`$f1ag` 呢？猜名字是碰运气，`GLOBALS` 是必中——**信息收集 > 暴力枚举**。

## 复盘

- 解题路径：读源码 → 认出 `$$` 可变变量 → `?args=GLOBALS` 一锅端 → 从 dump 里捞 flag1，Web 题第一次"纯浏览器通关"，零工具依赖
- **Web 方向首战**：从"拿到文件想办法"（MISC/Crypto）切换到"读懂代码找破绽"（代码审计）——思维方式从**构造解法**变成**发现漏洞**
- 新语法点一次记牢：`$$` 可变变量（变量名动态化）、`$GLOBALS`（全局变量仓库）、`/^\w+$/`（白名单正则）、`eval()`（危险函数警报）
- 破绽哲学：正则把非法字符全堵了，但 `GLOBALS` 是合法输入——**漏洞不一定是"攻破"，也可以是"防御没想清楚自己防的是什么"**
- 知识链回顾：linux/Linux2（认文件）→ telnet（抓流量）→ 眼见非实（拆容器）→ **变量1（审代码）**——MISC 三件套之后，Web 大门正式打开：变量1 是 PHP 代码审计系列（变量1 → …）的第一级台阶，后面的 Web 题将陆续遇到更多 PHP 危险函数与绕过姿势
- 下一题预告：同类型的 `eval`、`preg_match`、超全局数组还会反复出现，这道题的"看到 eval + 用户输入 = 有洞"条件反射值得练成肌肉记忆

## 参考

- [PHP 手册：可变变量](https://www.php.net/manual/zh/language.variables.variable.php)
- [PHP 手册：$GLOBALS](https://www.php.net/manual/zh/reserved.variables.globals.php)
- [PHP 手册：eval()](https://www.php.net/manual/zh/function.eval.php)
