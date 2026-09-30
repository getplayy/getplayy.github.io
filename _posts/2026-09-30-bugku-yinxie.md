---
layout: post
title: "【Bugku CTF】隐写 Writeup"
date: 2026-09-30 23:58:00 +0800
categories: ctf
---

- **平台**：[Bugku CTF](https://ctf.bugku.com/)
- **分类**：MISC
- **分值**：入门
- **考察点**：PNG 宽高隐写——IHDR 尺寸造假 + IDAT 数据量反推真身

> 个人学习记录，flag 已打码，请自行做题获取。

## 题目描述

附件解压得到一张 PNG。打开看着正常，但数据量"超重"——**图片下半部分被出题人用改小高度的方式藏起来了**。本题在沙箱中用 Python 完成全程取证。

## 解题过程

### 0x01 验明正身：头尾都健康

```python
data = open('2.png','rb').read()
w, h = struct.unpack('>II', data[16:24])   # IHDR 偏移 16~24
```

魔数 `89 50 4E 47 0D 0A 1A 0A` 正常、`IEND` 结尾正常、IHDR 声明 **500×420**、颜色类型 6（RGBA）。表面一切正常——**正常才是最大的异常**。

### 0x02 反推真实高度：IDAT 数据量泄了底

PNG 的像素数据存在 IDAT 块里（zlib 压缩）。**每行像素的字节数是固定的**：

```text
每行原始字节 = 宽 × 每像素字节数(RGBA=4) + 1(过滤码) = 500×4+1 = 2001
```

把所有 IDAT 拼接、解压，量一量总字节数：

```python
raw = zlib.decompress(idat)
true_h = len(raw) // 2001   # 1000500 // 2001 = 500
```

**声明 420 行，数据却够画 500 行**——高度被砍掉 80 行，flag 就藏在那 80 行里。

### 0x03 修复手术：改高度 + 重算 CRC

```python
data[20:24] = struct.pack('>I', 500)                      # 高度 420→500
data[29:33] = struct.pack('>I', zlib.crc32(data[12:29]))  # 重算 IHDR 校验和
```

CRC 不重算的话，严格的解码器会拒绝打开。修复后用 PIL 渲染：**500×500 完整图现形**——底部多出一条黑带。

### 0x04 黑带里的字看不清？透明通道才是正主

RGB 视角下底部黑带的文字糊成一片。关键一步：**这图是 RGBA**——把 alpha 通道单独可视化（非全不透明的像素标黑），瞬间清晰：

- 图中央浮出 `Bugku...` 半透明水印
- **底部一行白色大字：BUGKU{***}**

## 原理分析

### 宽高隐写：骗显示器，不骗数据

PNG 的显示尺寸由 IHDR 里的宽/高字段**声明**，而像素数据老老实实存在 IDAT 里。改小高度后：

- 图片查看器只渲染前 420 行 → 下半部分"存在但不可见"
- **数据一个字节没少**——这就是破绽

所以有两个取证入口：直接改高度暴力看全图（通用），或者像本题一样**用 IDAT 数据量反推真实尺寸**（精准）。反推公式：

```text
真实高度 = 解压后总字节数 ÷ (宽 × 每像素字节数 + 1)
```

### CRC：PNG 的每块都有"防伪码"

PNG 按"块"组织（IHDR/IDAT/IEND……），每块尾部 4 字节 CRC 校验。**改了块内容必须重算 CRC**，否则解码器报错——这一步是十六进制手工改图最容易翻车的地方，Python 里 `zlib.crc32` 一行搞定。

### 隐写思路谱系 +1

```text
改后缀（眼见非实）→ 解压搜字符串（easy_nbt）→ 改宽高（本题）→ LSB/盲水印（进阶待学）
```

共同心法：**文件的"显示层"和"数据层"是两回事**——眼见不为实，数据量才是真相。

## 复盘

- 解题路径：验头尾 → IDAT 量出真实高度 500 → 改 IHDR + 重算 CRC → alpha 通道看 flag，全程 Python 十几行
- **新姿势入库**：PNG 块结构（长度+类型+数据+CRC）、IHDR 偏移 16=宽/20=高、`zlib.crc32` 重算校验、RGBA 的 alpha 通道单独看图
- **踩坑实录**：① RGB 视角看 RGBA 图，文字糊在黑带里——**隐写题遇 RGBA 先拆通道**；② 导出中间图时 convert('RGB') 把 alpha 丢了，绕了个弯——每一步转换都在丢信息
- **沙箱 Python 取证流**成型：下载附件 → 结构解析 → 修改 → 渲染可视化，一个脚本闭环，比十六进制手工改图稳得多
- 知识链回顾：linux/Linux2（认文件）→ 眼见非实（拆容器）→ easy_nbt（解 gzip）→ source（挖历史）→ **隐写（改尺寸）**——"文件结构解剖"技能树点亮第四层
- 下一题预告：宽高只是隐写的入门级，后面还有 LSB 最低位隐写（zsteg/stegsolve）、盲水印（BlindWaterMark）、文件附加（binwalk 分离）——MISC 隐写家族全明星还在后面

## 参考

- [PNG 规范（RFC 2083）：块结构与 CRC](https://www.rfc-editor.org/rfc/rfc2083)
- [Pillow 文档：Image 模块](https://pillow.readthedocs.io/en/stable/reference/Image.html)
- [CTF Wiki：图片隐写](https://ctf-wiki.org/misc/picture/introduction/)
