# [BugKu CTF] MISC —— linux & Linux2 Writeup

> **题目方向**：杂项（MISC）
> **题目作者**：harry
> **难度**：入门 ⭐
> **涉及知识点**：Linux 基础命令（tar / file / cat / tail / strings / grep）、二进制文件中的字符串提取、文件类型识别

BugKu MISC 区有两道连号姊妹题：**linux**（15 分）和 **Linux2**（25 分）。考点完全同源——"会不会用命令行在一堆二进制数据里捞字符串"，放在一份 writeup 里讲。

---

## 一、题目信息

### 1.1 linux（id=15）

- **描述**：linux基础问题
- **提示**：`key{}`
- **附件**：`1.tar.gz`

### 1.2 Linux2（id=19）

- **描述**：（无）
- **提示**：给你点提示吧：key的格式是`KEY{}`
- **附件**：`brave.zip`，解压得到一个**无后缀**的 `brave` 文件（约 20MB）

> 两道题的提示都在告诉你 flag 的**格式**——这本身就是解题钥匙，后面会用到。

---

## 二、解题思路

### 2.1 linux：解压 → 定位 → 捞字符串

**第一步：解压**

```bash
$ file 1.tar.gz
1.tar.gz: gzip compressed data, from Unix, original size modulo 2^32 10240

$ tar -zxvf 1.tar.gz
test/
test/flag
```

参数拆解：`-z`（用 gzip 处理）、`-x`（解包）、`-v`（显示过程）、`-f`（指定文件名）。解压出 `test` 文件夹，里面躺着一个**无后缀**的 `flag` 文件。

**第二步：识别文件类型**

```bash
$ file test/flag
test/flag: Linux rev 1.0 ext3 filesystem data  # 原题附件的输出
```

`file` 命令通过**文件头魔数（Magic Number）**判断真实类型——这文件其实是个 **ext3 文件系统镜像**（题目叫 "linux" 的由来）。做 CTF 拿到任何陌生文件，第一件事永远是 `file` 一下。

**第三步：在二进制数据里捞 flag**

直接 `cat flag` 会刷出一大屏乱码（ext3 镜像里全是二进制数据），但拉到**最后几行**就能看到明文的 `key{...}`：

```bash
$ tail flag          # 只看最后几行，比 cat 省事
```

更优雅的做法是让工具替你筛——**strings 提取所有可打印字符串，grep 过滤**：

```bash
$ strings test/flag | grep -i key
key{feb81d3834e2423c9903f4755464060b}
```

或者一步到位，让 grep 把二进制文件当文本搜（`-a` 参数）：

```bash
$ grep -a "key{" test/flag
```

> **没有 Linux 环境？** 两个等价替代：
> - 把 `flag` 文件后缀改成 `.zip`/`.rar`，用 7-Zip/WinRAR 一路解压，能解出一个 `flag.txt`，Ctrl+F 搜 `key`；
> - 用 010 Editor / WinHex 打开，文本搜索 `key{`。

### 2.2 Linux2：当心干扰项

解压 `brave.zip` 得到无后缀的 `brave` 文件。很多人顺手 `foremost brave` / `binwalk -e brave` 做文件分离，**确实分离出一张图片，图片上明晃晃写着 `flag{...}`——但提交是错的！** 这是出题人埋的干扰项。

回头看题目提示：**"key的格式是 KEY{}"**——注意是大写 `KEY`，不是 `flag`。图片上那个 `flag{...}` 格式根本对不上，早就该起疑。

正解和上一题同一个套路：

```bash
$ strings -a brave | grep "KEY{"
KEY{24f3627a86fc740a7f36ee2c7a1c124a}
```

Windows 下同样：010 Editor 打开 brave，文本搜索 `KEY`，直接出结果。

---

## 三、Flag

```
linux：  key{feb81d3834e2423c9903f4755464060b}
Linux2： KEY{24f3627a86fc740a7f36ee2c7a1c124a}
```

---

## 四、命令速查（本题用到 + 举一反三）

| 命令 | 作用 | 本题用法 | 备注 |
|---|---|---|---|
| `file` | 识别文件真实类型 | `file flag` | 拿到陌生文件的第一步 |
| `tar -zxvf` | 解压 .tar.gz | `tar -zxvf 1.tar.gz` | `-x`解包 `-z`gzip `-v`显示 `-f`文件名 |
| `cat` / `tail` / `tac` | 查看 / 看尾 / 倒序看 | `tail flag` | 二进制文件慎用 cat |
| `strings` | 提取可打印字符串（默认≥4字符） | `strings -a brave` | 二进制取证三杰之一 |
| `grep -a` | 把二进制文件当文本搜索 | `grep -a "key{" flag` | 不加 `-a` 只报 "Binary file matches" |
| `grep -i` | 忽略大小写 | `grep -i key` | 大小写不确定时用 |
| `grep -rE` | 递归 + 扩展正则 | `grep -rE "KEY\{\|flag\{" dir` | 全目录扫 flag 的惯用命令 |
| `foremost` / `binwalk -e` | 按文件头分离隐藏文件 | （本题是坑） | 分离出 ≠ 正确答案 |

组合拳套路（值得背下来）：

```bash
strings -a 文件名 | grep -iE "key\{|flag\{"    # 万能字符串打捞
grep -rE "flag\{|key\{|KEY\{" .                # 递归扫整个目录树
```

---

## 五、踩坑记录

1. **`cat` 刷屏 + 终端乱码**：对二进制文件 `cat`，轻则刷屏，重则终端字符集被带歪（满屏乱码）。恢复方法：输 `reset` 回车（输入时看不见也照敲）。以后看二进制文件优先 `strings` / `tail` / `less`。
2. **grep 遇到二进制文件"装哑巴"**：不加 `-a` 时 grep 只输出 `Binary file xxx matches`，不显示匹配内容——新手常在此卡住，以为没搜到。
3. **Linux2 的图片是干扰项**：工具分离出的文件 ≠ 答案。**判据就是题目给的 flag 格式**——图片上写的是 `flag{...}`，题目要的是 `KEY{...}`，格式对不上就该立即回头。这题 25 分的"含金量"全在这个陷阱上。
4. **大小写是格式的一部分**：`key{}` 和 `KEY{}` 是两个不同题目的两种格式，提交时严格照题目来。搜索时可以 `grep -i` 宽进，提交时严出。
5. **别被 20MB 的 brave 吓到**：文件大不代表要"看完"它，strings + grep 秒级定位——工具思维，不是人肉思维。

---

## 六、知识点总结

### 6.1 字符串打捞：MISC 的基本功

CTF 里大量题目最终归结为一句话：**"flag 就藏在文件的某个角落，把它找出来"**。本题的 flag 直接以明文嵌在二进制数据里，解法分层：

1. **肉眼层**：`cat` / `tail` / 十六进制编辑器翻看（费眼，最后兜底）；
2. **工具层**：`strings | grep`（本题正解）；
3. **自动化层**：`grep -r` 递归扫目录、写脚本批处理（文件多时）。

### 6.2 文件类型识别

`file` 命令读的是**魔数**：ext3 镜像、PNG 的 `89 50 4E 47`、ZIP 的 `PK`（`50 4B`）……这是隐写题、取证题的入口动作。本题 `flag` 文件实际是 ext3 镜像，还能再进一步玩：

```bash
mkdir /mnt/ext3
mount -o loop test/flag /mnt/ext3   # 挂载镜像，浏览"整个文件系统"
```

挂载后能像普通目录一样浏览它——这是**磁盘取证**的标准姿势。

### 6.3 出题人视角：提示即钥匙

两道题的提示都只说了**格式**（`key{}` / `KEY{}`）。这不是废话——它同时给了你三样东西：搜索关键字、结果校验标准、干扰项过滤器。以后做题先把题目提示里能提取的关键词列出来再动手。

---

## 参考

- [BugKu 题目页 - linux](https://ctf.bugku.com/challenges/detail/id/15.html)
- [BugKu 题目页 - Linux2](https://ctf.bugku.com/challenges/detail/id/19.html)

> **一点感想**：这两题是纯"工具熟悉度"考题——没有加密、没有算法，考的就是 Linux 命令行肌肉记忆。`strings + grep` 这套组合拳你会在本平台后面几十道 MISC/Forensics 题里反复用到，把它练到不用过脑子，这两题的价值就赚回来了。
