---
layout: post
title:  "Git Internal - 读取 Git 中的索引区"
date:   2015-10-01 14:07:00
categories: git_internal
---

在我的上一篇 Blog [《Git Internal - Git 中的索引区 Index 的概念》](/git_internal/2015/10/01/git-index-concept.html) 中已经提到，Git 并没有命令可以直接看到索引区的内容，因此给人不直观的感受，在这篇 Blog 中，我将给出读取 Git 索引区内容的方法。

在 Git 中，索引区的内容全部存储在 `.git/index` 中，但这个文件除了存储 Git 索引区之外，还有另外一个功能，就是充当当前 Git Repo 的文件锁。在 Git 中，对于所有会读取或写入 Index 的命令，都会在执行时先讲 `.git/index` 的内容全部读入内存，然后创建 `.git/index.lock`，并将更新后的索引信息写入到这个文件中，直到命令结束前，才将 `.git/index.lock` 重命名为 `.git/index`，覆盖原来的索引区内容。如果此时有另外一条命令也试图读取或修改索引区，就会检测到 `.git/index.lock` 的存在，然后拒绝执行，这样相当于保证了 Git Repo 不会出现因为竞争和冲突造成的错误。 在 Git 中，类似的机制也存在于其他一些文件，例如 object 文件，但是只有 index 的锁才能锁住整个 Git Repo，因此格外重要。

打开 `.git/index` 文件，首先要做的就应该是检查文件的正确性，检查分为三步：

1. 确定文件确实是一个索引文件
2. 确定索引文件的版本是当前 Git 版本可以处理的
3. 确定文件本身没有损坏

在 `.git/index` 中，开头的 12 个字节是索引文件的元信息，它们从前往后分别是索引文件的签名，版本以及索引中条目的数量，每项各占 4 个字节。由于 Git 被设计成了文件结构是跨平台的，因此所有数字在文件中都是以网络序存储，即 big endian 存储。对于 Intel x86 平台，必须记得用 `ntohl` 函数去将它转换为 little endian 的格式才能读取。

所有索引文件的签名都是统一的，换成 ASCII 字符的话内容是 DIRC，如果转成网络序数字的话就是 0x44495243。对于当前版本的 Git，能处理的索引文件版本都是 2～4，一般以 2 为主。本文也会以版本 2 作为案例来解释索引文件的结构。

检查文件本身完整性的方法是 Git 中随处可见的 SHA-1，方法是对整个索引文件，除了最后 20 字节之外的内容计算 SHA-1，并与最后 20 字节的内容相比较。Git 生成索引文件时会计算文件内容的 SHA-1 并追加在文件的最后部分。

在 Git 中，SHA-1 既有由 20 字节构成的二进制版本，也有由 40 字节的十六进制版本，二者完全等效，但是 Git 在不同的情况下会挑选不同版本的呈现方式来使用。索引文件由于是二进制文件，所以始终挑选 20 字节的二进制版本来存储，相对更加节省存储空间。

索引文件的主体部分是由一项项索引条目构成，每个条目代表一个文件的元信息，大小不定，但至少有 62 个字节，条目数量可以在索引文件的元信息中获取。

每个条目从前往后的内容是这样的（所有数字都以网络序存储）：

1. 4 个字节的 ctime 的时间戳的整数部分
2. 4 个字节的 ctime 的时间戳的小数部分，精确到纳秒。
3. 4 个字节的 mtime 的时间戳的整数部分，
4. 4 个字节的 mtime 的时间戳的小数部分，精确到纳秒。
5. 4 个字节的设备号
6. 4 个字节的 inode 号
7. 4 个字节的文件权限
8. 4 个字节的 uid
9. 4 个字节的 gid
10. 4 个字节的文件大小
11. 20 个字节的 SHA-1
12. 2 个字节的标志符
13. 2 个字节的扩展标志符［可选］
14. 不定长字节的文件名

如果标志符的 0x4000 位置被标记，则之后的两个字节为扩展标志符，将添加到标志符的高位部分，这样标志符就有 4 个字节构成。

标志符的低 6 个字节表示条目中文件名的长度，这样每个条目可以记录最多 64 个字节的文件名，但是由于很多操作系统的文件名长度上限都不会这么短，当超过或等于 64 个字节后，Git 会改变方法，通过和 C 字符串一样的 `\0` 标志来标示文件名的结束，同时标志符的低 6 个字节全部置 1 （0x0fff）。

索引区文件的索引条目之间并不紧密连接，而是按照 8 个字节对齐。

我已经在 GitCafe 上创建了一个项目 [Play Git](https://gitcafe.com/bachue/play_git)，里面的 `read_index.rb` 可以读取并显示索引文件的内容，可以通过 `ruby read_index.rb .git/index` 来执行，有兴趣的话可以去试试看。