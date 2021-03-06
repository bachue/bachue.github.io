---
layout: post
title:  "Git Internal - Git 中的索引区 Index 的概念"
date:   2015-10-01 12:49:00
categories: git_internal
---

说起 Git 的索引区，原本是希望在由我编写并拍摄的[《Git 实用教程》](http://www.stuq.org/course/detail/1026)中的最后一课提到的，我甚至已经做了一个动画演示来详细解释索引区的概念，但是由于我们的 CTO [@Rainux](https://gitcafe.com/rainux) 认为索引区涉及内部实现，难度过大，很多非技术人员未必能理解，于是又将这些内容全部移除。

为了便于对下文的理解，这里我先简单讲下索引区的概念。

索引区在 Git 中是以一个 key-value 形式的二进制文件存在，它的 key 是 Git Repo 中每一个被跟踪的 Git 文件的路径，value 中则包含该文件内容的 SHA-1 及其他相关的元信息。

当文件被修改时，文件的 SHA-1 自然发生改变，就会与索引区的记录不符合，此时 Git 就知道这个文件发生了修改，并且这个修改是 Unstaged 状态。

一旦用了 `git add` 或是底层命令 `git update-index`，那么 Git 会计算新的文件内容的 SHA-1，为它创建出 blob 对象，然后将新的 SHA-1 更新到索引区里，这样索引区的记录就和 working tree 中文件的 SHA-1 符合，但是由于当前分支引用的 commit 还没有改变，因此 commit 中记录的该文件的 SHA-1 与索引区的记录不符合，因此 Git 认定这个修改处于 Staged 状态。

一旦用了 `git commit` 或是底层命令 `git commit-tree`，那么 Git 创建新的 commit 对象，并且将索引区里的 SHA-1 记录在 commit 对象下的 tree 对象中，这样当前分支引用的 commit - 索引区 - working tree 中该文件的 SHA-1 全都一致，该文件的修改就会被认定为已经提交。

其实索引区的概念本身并不复杂，只是由于 Git 并没有什么命令能直接看到索引区的记录（`git ls-files -s --debug` 已经是最接近的命令了，但依然不能完整体现索引区的内容），因此很难给人直观的感受吧。

不过我之所以想要在一个面向新手的教程中提及索引区的概念，是因为在 Git 中，很多命令的功能的精确描述就与索引区有关，例如：

用 `git reset` 将当前分支设置到指定的 commit 时，默认情况下用的是 `--mixed` 选项，该选项在 man 中的描述是这样的：

> Resets the index but not the working tree (i.e., the changed files are preserved but not marked for commit) and reports what has not been updated.

设置索引区但是不修改 working tree 中文件的内容，因此可以知道该文件将处于 Unstaged 状态。

而 `--soft` 选项的描述是：

> Does not touch the index file or the working tree at all (but resets the head to <commit>, just like all modes do).

不设置索引区和 working tree 中文件的内容，commit 中的文件 SHA-1 与索引区不符合，属于 Staged 状态。

而 `--hard` 选项的描述则是：

> Resets the index and working tree. Any changes to tracked files in the working tree since <commit> are discarded.

同时设置索引区和 working tree 中的文件的内容，这样当前分支引用的 commit - 索引区 - working tree 中该文件的 SHA-1 全都一致，因此执行 `git status` 将看不到这个文件之前的情况，之前所有针对该文件的修改都将丢失。

另一个与索引区相关的命令时 `git checkout`，它的描述是

> Updates files in the working tree to match the version in the index or the specified tree.

对于给出了文件路径，但没有指定 commit 的情况，`git checkout` 将文件内容修改成与索引区里的记录相符合。这样就会造成所有针对该文件的 Unstaged 的修改会被取消，而 Staged 的修改则毫发无损。

可以看到，使用索引区来解释 Git 命令的功能会比一般的通俗解释更佳清晰明白，但是需要观众完全理解 Git objects 和 索引区的概念并且还要费点脑筋。

所以后来还是在教程中采用了通俗易懂的（但不精确的）概念来描述 `git reset` 和 `git checkout` 的含义，而将精确的描述留在了这篇 Blog 中。

如果您想了解 Git 索引区内部更多实现细节，可以阅读我的下一篇 Blog [《Git Internal - 读取 Git 中的索引区》](/git_internal/2015/10/01/git-index-internal.html)。
