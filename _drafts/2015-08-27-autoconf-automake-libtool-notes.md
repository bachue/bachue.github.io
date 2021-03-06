---
layout: post
title:  "Autotools - A Practioner's Guide to GNU Autconf, Automake, and Libtool 读书笔记"
date:   2015-08-27 21:24:00
categories: unix
---

## AutoConf

### AC_PREREQ(version)

检查 Autoconf 最低版本，这是唯一可以写在 `AC_INIT` 之前的宏（意思是这一般是写在 configure.ac 中的第一个宏）

### AC_INIT(package, version, [bug-report], [tarname], [url])

初始化 Authconf 系统，其中 `bug-report`， `tarname`， `url` 三个参数可选。默认情况下，Autoconf 声称的 tar 包的文件名是 `package-version.tar.gz`。
`bug-report` 一般是 Email 地址，但也可以是其他内容。`version` 不规定版本的格式，任何文字都可以是版本。`url` 则通常是项目的网址，通过 `configure --help` 可以看到。

### AC_CONFIG_SRCDIR(unique-file-in-source-dir)

这个宏其实只是一个防呆测试，防止 `configure` 命令找到错误的源代码目录（例如用户传入了错误的 `--srcdir` 参数）。
因此，该方法尝试搜索源码目录中一个独有的源码文件，通过检测它的存在性来确定源码目录是否正确。

### AC_CONFIG_XXXS(tag..., [commands], [init-cmds])

Autoconf 有四种实例化宏，分别是 `AC_CONFIG_FILES`，`AC_CONFIG_HEADERS`，`AC_CONFIG_COMMANDS`，`AC_CONFIG_LINKS`，此类宏接受一组文件或者标签。`configure` 脚本负责为他们通过模版生成实际的文件
（有些老版本的 Autconf 还在用 `AC_CONFIG_HEADER` 宏，这个宏没有它的复数版本更强大）。
例如 `AC_CONFIG_HEADERS([config.h])`，它可以被展开成为 `AC_CONFIG_HEADERS([config.h:config.h.in])`，即 `config.h` 通过模版文件 `config.h.in` 生成。
也可以写 `AC_CONFIG_HEADERS([config.h:cfg0:cfg1:cfg2])`，那么 `config.h` 将由 `cfg0`，`cfg1`，`cfg2` 三个模版文件按照顺序合成。

### AC_CONFIG_COMMANDS

这个实例化宏比较特别，它并不生成任何文件而是执行任意的 Shell 语句，但是它会被生成的 `config.status` 自动执行，或者也可以让 `config.status` 执行其中某个命令。
例如只要写了 `AC_CONFIG_COMMANDS([abc], [echo "Testing $mypkgname"], [mypkgname=$PACKAGE_NAME])`，
每次执行 `config.status` 的时候会自动执行 `abc` 命令来显示 `Testing test` 的字样。
可以通过 `config.status --help` 看到 `config.status` 支持 `abc` 这样一条命令。因此通过执行 `config.status abc` 也可以直接显示 `Testing test` 的字样。

### AC_CONFIG_SUBDIRS(dir1[ dir2 ... dirN])

声明子项目的所在目录

### AC_CONFIG_MACRO_DIR(macro-dir)

声明一个目录包含所有由 `aclocal` 生成的 aclocal.m4 文件，供 `autoconf` 读取。

### AC_CHECK_HEADERS(header-file..., [action-if-found], [action-if-not-found], [includes = 'default-includes'])

`AC_CHECK_HEADERS` 会通过 `autoheader` 生成 `config.h.in`，增加多个 `undef` 宏的语句。
例如 `AC_CHECK_HEADERS([unistd.h foobar.h])` 会生成这样的 `config.h.in`：

{% highlight c linenos %}
#undef HAVE_UNISTD_H
#undef HAVE_FOOBAR_H
{% endhighlight %}

随即执行 `configure` 可以会检查 `unistd.h` 与 `foobar.h`，一般来说，前者总是存在的而后者并不存在，因此会得到这样的 `config.h`：

{% highlight c linenos %}
#define HAVE_UNISTD_H 1
/* #undef HAVE_FOOBAR_H */
{% endhighlight %}

可以看到会自动生成 `HAVE_UNISTD_H` 这样一个宏，表示 `unistd.h` 已经找到，而 `HAVE_FOOBAR_H` 没有生成是因为 `foobar.h` 没有找到。可以在源码中 include `config.h` 来使用这些宏判定某些头文件是否存在，从而灵活应对，实现跨平台性。

### AC_SUBST(shell_var[, value])

定义模版文件中的 Autoconf 变量（不要和模版文件本身的变量混淆），在模版文件中通过前后增加 @ 来调用。

### AC_DEFINE(variable) / AC_DEFINE(variable, value[, description]) / AC_DEFINE_UNQUOTED(variable) / AC_DEFINE_UNQUOTED(variable, value[, description])

定义 C 预编译器的宏，AC_DEFINE 与 AC_DEFINE_UNQUOTED 功能相似，只是 AC_DEFINE_UNQUOTED 会利用 Shell 的语法展开 `value`。
由于 C 预编译器的宏可以有值也可以只有定义，因此 AC_DEFINE 也分成了两个版本。
至于 `description` 只是在 `autoheader` 生成的 `config.h.in` 的注释中会包含。

### AC_PROG_XXX([compiler-search-list])

此类宏用语检查某个工具的存在性，例如 `AC_PROG_CC` 检查 C 编译器的存在。可以传入多个参数用以检查多个编译器，例如 `AC_PROG_CC([cc cl gcc])`，也可以省略参数由 `configure` 自己搜索。
类似的宏还有 `AC_PROG_INSTALL`，`AC_PROG_INSTALL`，`AC_PROG_LEX`，`AC_PROG_YACC`，`AC_PROG_SED`，`AC_PROG_AWK` 等等。此类宏不仅会检查对应的工具的存在性，还会生成相应的 `Autoconf` 变量可以用在 `Makefile.in` 中。
`autoscan` 在生成 `configure.scan` 的时候也会根据源码中的文件后缀名来选择应该生成的宏去检查相应的工具的存在。

### AC_PROG_RANLIB

只要有任何库被编译，就会要求使用这个宏。

### AC_CHECK_PROG(variable, prog-to-check-for, value-if-found, [value-if-not-found], [path], [reject])

自定义检查宏，检查 `prog-to-check-for` 的存在性，如果存在，设置 `variable` 为 `value-if-found`，否则设置为 `value-if-not-found`，此外 `value-if-found` 或 `value-if-not-found` 如果被设置，也会被显示在 `configure` 的输出中。`reject` 相当于黑名单，通常用绝对路径。

### AC_SEARCH_LIBS(function, search-libs, [action-if-found], [action-if-not-found], [other-libraries])

检查依赖库宏，通过编译一个测试程序来测试库的存在，`function` 是需要被检查的函数，`search-libs` 是搜索用的库，如果成功，执行 [action-if-found]，否则执行 [action-if-not-found]，有些库可能还要依赖其他库才能编译成功，这样的库可以写在 `other-libraries` 中，空格分割，例如 `'-lXt -lX11'`

### AC_MSG_CHECKING, AC_MSG_RESULT, AC_MSG_NOTICE, AC_MSG_WARN, AC_MSG_ERROR, AC_MSG_FAILURE

`AC_MSG_CHECKING` 和 `AC_MSG_RESULT` 配对使用，`AC_MSG_CHECKING` 用来输出检查的目标，不会回车。而 `AC_MSG_RESULT` 用来输入检查的结果，使用后自动回车。
`AC_MSG_NOTICE` 和 `AC_MSG_WARN` 接近，分别输出普通信息和警告信息。`AC_MSG_ERROR` 和 `AC_MSG_FAILURE` 生成错误信息，后者还会输出当前目录路径，这两个宏都接受第二个参数作为命令返回值。

### AC_ARG_WITH(package, help-string, [action-if-given], [action-if-not-given])

参数宏，检查参数中是否有 `--with-xxx` 或者 `--without-xxx` 出现，如果有，执行 `action-if-given`，否则执行 `action-if-not-given`

### AC_ARG_ENABLE(feature, help-string, [action-if-given], [action-if-not-given])

参数宏，检查参数中是否有 `--enable-xxx` 或者 `--disable-xxx` 出现，如果有，执行 `action-if-given`，否则执行 `action-if-not-given`

### AS_HELP_STRING(left-hand-side, right-hand-side, [indent-column='26'], [wrap-column='79'])

`AC_ARG_WITH` 和 `AC_ARG_ENABLE` 的辅助宏，用来生成 `help-string`，`left-hand-side` 传入参数而 `right-hand-side` 传入帮助信息，`AC_HELP_STRING` 生成帮助用的字符串，可以按照默认或自定义的缩进和限长。

### AC_CHECK_TYPES(types, [action-if-found], [action-if-not-found], [includes = 'default-includes'])

检查类型是否存在，如果存在，就执行 `action-if-found`，否则执行 `action-if-not-found`

### AC_OUTPUT

用来为 `configure` 生成 `config.status` 的宏，总是作为 `configure.ac` 中的最后一个宏。

## AutoMake

### configure.ac

#### AM_INIT_AUTOMAKE

`AM_INIT_AUTOMAKE` 是 AutoMake 的宏，用来生成自动生成 Makefile 的脚本。使用了这个脚本后，执行 `autoreconf -i` 会执行 `automake -i`，`automake` 会生成一系列实用工具文件，`aclocal.m4`，`install-sh`，`missing`，`depcomp`，`INSTALL`，`COPYING`，其中 `INSTALL` 是默认的安装步骤而 `COPYING` 则是 GPLv3 协议。

### Makefile.am

#### SUBDIRS =

指出包含 Makefile 的子目录。

#### CLEANFILES =

指出所有应该被 `make clean` 删除的文件。

#### EXTRA_DIST

需要被额外发布的文件

#### Prefixes

##### bin_

需要被生成的可执行程序

##### lib_

需要被生成的库

##### noinst_

指出所有需要被 build 但不需要被安装的程序。

##### check_

指出所有只需要被 `make check` build 用于测试单不需要被安装的程序。

##### EXTRA_

指出所有可以被条件 build 的程序

#### Primaries

##### _PROGRAMS =

生成编译器和链接器 build 可执行程序的规则。

##### _LIBRARIES / _LTLIBRARIES

生成编译静态库的规则，其中 `_LTLIBRARIES` 也会生成 build libtool 共享库和执行这些工具的规则。

##### _JAVA / _PYTHON

分别生成编译 Java 和 Python 的规则。

##### _DATA

任意数据文件

##3## _HEADERS

头文件

##### _MANS

用 UTF-8 编码的 troff 文件，会被 man 渲染然后呈现给用户。

#### POV

##### product_LDADD

额外的 object 文件

##### product_CPPFLAGS

C 预编译器选项

##### product_CFLAGS

C 编译器选项

##### product_LDFLAGS

链接器选项

##### library_LIBADD

非 libtool 链接器对象和静态链接库

##### ltlibrary_LIBADD

libtool 链接器对象 .lo 和静态链接库 .la

#### Modifiers

##### dist

将被 `make dist` 包含在发布的包里的文件

##### nodist

将不被 `make dist` 包含在发布的包里的文件

##### nobase

将在发布的文件里保留文件路径

##### notrans

对于 man 的 PLV 文件将不需要被改名（即不需要在安装时将 .man 文件改名为 .N(0, 1, 2)）

## Libtool

#### LT_INIT(options)

初始化 libtool，接受 dlopen，disable-fast-install，shared，disable-shared，static，disable-static，pic，no-pic 等选贤。

