---
layout: post
title:  "Git Internal - Git 中的 Object 和 Reference"
date:   2015-10-01 09:23:00
categories: git_internal
---

Git Internal 将作为一个系列来探究 Git 中文件的存储格式和网络协议，所有对 Git 有兴趣的同学都可以参考和学习，探究存储格式或网络协议并不一定能提高大家使用 Git 的水平，但却可以帮助大家将 Git 命令的原理和使用了解的更加清晰。

正如在由我编写并拍摄的《Git 实用教程》中提到的，Git 将 Repo 中所有数据，包括提交信息，子模块信息，所有引用，所有引用日志都放在 .git 目录中。本文主要探究 .git 目录下的 objects 目录和 refs 目录中的内容。

一般情况下 `refs/` 目录下会通过文本文件的方式存储所有本地分支（`heads/`），远程跟踪分支（`remotes/`）和标签（`tags/`）的信息，存储格式非常简单，一种是直接记录 commit 的 SHA-1 的十六进制格式，大部分 refs 下的文件都会采用这个方法记录它们引用的 commit。另一种是引用另一个文件，以 ref: 开头，后面跟 .git 里文件的相对路径，通常 .git/HEAD 会使用这种方法，使当前分支可以引用另一个分支所引用的 commit，如果这个分支引用的 commit 发生了改变，当前分支 HEAD 也会随之改变。由于引用全部存储在 `refs/` 目录下，你也可以自己创建或编辑文件来设置 branch 或者 tag，只是那并不是最安全的方法，推荐使用 Git 底层命令 `git update-ref` 来直接编辑引用。

`objects/` 目录下存储 Git 所有对象文件，对于每个 object，都以它的 SHA-1 的前两位为所在目录，后 18 位为文件名，相当于建立了一层简单的索引。Git 一共有四种对象，commit，tree，blob 和 annotated tag。它们的关系是这样的，annotated tag 记录 commit 的 SHA-1，commit 记录 tree 对象的所在位置，tree 是 Git 中一个目录的所有 entry 的集合，对于每一个 entry，如果是文件，记录文件的 blob 对象的 SHA-1，如果是目录，记录该目录的 tree 对象的 SHA-1，如果是子模块，记录子模块的 commit 对象的 SHA-1。blob 对象记录所有被 Git 跟踪的文件的内容。

每个 object 文件都会被 deflate 方法压缩，解压后，数据的一开始是自身的类型名称，名称也只有四种，commit，tree，blob，tag，随后空一格，然后紧跟数据部分的长度的字符串形式，之后会有一个 `\0` 字符分割元信息和后面的数据主体部分，例如 `tree 313\x00`，`tag 132\x00` 分别表示一个长 313 字节的 tree 对象和一个长 132 字节的 annotated tag 对象。

至于数据的主体部分，不同的 object 类型有不同的格式，查看 Git Object 内容的命令是 `git cat-file -p [refs or sha-1]`。首先先来看下 annotated tag 的格式：

{% highlight text linenos %}
object [Object SHA-1 十六进制]
type [Object type]
tag [Tag name]
tagger [Author Name] <[Author Email]> [Timestamp] [Timezone]

[Tag message]
{% endhighlight %}

可以看到 annotated tag 格式的可读性很强，每项属性按行分割，key 和 value 之间按空格分割，最后部分则是不限长的 tag message 部分，设计合理，简明易懂。

注意，仅仅创建 annotated tag 才会创建对应的 object，如果只是普通的 tag 是没有 object 被创建的，仅仅只有 refs 被创建。另外，创建 annotated tag 也会在 refs 中创建对应的文件，否则 `git tag` 是找不到这个 tag 的。

commit 的格式与 annotated tag 十分相似：

{% highlight text linenos %}
tree [Tree SHA-1 十六进制]
parent [Commit SHA-1 十六进制]
[More parents]
author [Author Name] <[Author Email]> [Timestamp] [Timezone]
committer [Author Name] <[Author Email]> [Timestamp] [Timezone]

[Commit message]
{% endhighlight %}

注意 parent 是可以有不止一行的，对于 merge commit，parent commit 至少就有两个。

Git 通过 commit 可以找到 Git Repo 根目录的 tree 对象，tree 对象是 Git object 中少有的二进制格式（`git cat-file -p` 会美化 tree 对象的显示结果，使之变成可读的文本），里面记录了一层目录中所有 entry 的信息，它的格式是：

{% highlight text linenos %}
[八进制文件权限] [文件名]\x00[Blob, Commit or Tree SHA-1 二进制]
{% endhighlight %}

每一项 entry 之间没有任何标志连接，那是因为所有 SHA-1 的二进制格式就是 20 个字节，无需通过其它手段分割。所有 entry 都会通过文件名排序。

至于 blob 是最简单的，里面直接记录了整个文件的内容，注意，在 Git 使用本文所介绍的方法记录数据时，Git 不会使用 patch 技术，即使只是对文件作出微量的修改，也会导致一个包含文件新版本完整内容的 blob 对象被创建，因此对容量会有一定的浪费。在之后介绍打包存储格式时还会有两种 patch 技术被引进，会大幅降低存储浪费，同时读取更为高效，敬请期待。

我已经在 GitCafe 上创建了一个项目 [Play Git](https://gitcafe.com/bachue/play_git)，里面的 `read_object.rb` 可以读取并解析 object 文件的内容使之成为 Ruby 对象，可以通过 `ruby read_object.rb .git/objects/[object file]` 来执行，有兴趣的话可以去试试看。
