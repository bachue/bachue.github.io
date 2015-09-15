---
layout: post
title:  "Ruby 执行 Shell 命令经验谈"
date:   2015-09-15 22:12:00
categories: jekyll update
---

# Ruby 执行 Shell 命令经验谈

如果你 Google "Ruby 执行 shell" 就可以看到有篇文章排名第一 [《Ruby 执行 Shell 命令的六种方法》](http://droidyue.com/blog/2014/11/18/six-ways-to-run-shell-in-ruby/)，这篇文章介绍了 Ruby 中执行 Shell 的六种方法：

### exec

exec 会将指定的命令替换掉当前进程中的操作，指定命令结束后，进程结束。

Ruby 代码：

{% highlight ruby linenos %}
exec 'echo "hello world"'
print 'abc'
{% endhighlight %}

输出：

{% highlight text linenos %}
hello world
{% endhighlight %}

### system

system 会返回布尔值来表明命令执行结果是成功还是失败。

Ruby 代码：

{% highlight ruby linenos %}
system 'echo "hello $HOSTNAME"'
#=> true
{% endhighlight %}

输出：

{% highlight text linenos %}
hello MacBook-Pro-2.local
{% endhighlight %}

### 反引号（`）

反引号借鉴于 Shell，语义也一致，获取命令从 STDOUT 输出的内容，是 Ruby 中最简单的命令行调用方法。

Ruby 代码：

{% highlight ruby linenos %}
today = `date`
=> "Tue Sep 15 14:04:30 CST 2015\n"
$?
=> #<Process::Status: pid 23359 exit 0>
$?.to_i
=> 0
{% endhighlight %}

此外还有三种方法掌握的人并不多

* IO.popen
* Open3.popen3
* Open4.popen4

这样看起来 Ruby 中调用 Shell 还是蛮简单的，于是就有了这样（ worst practices ）的代码：

* 通过将密码嵌入字符串的方式执行 Shell

{% highlight ruby linenos %}
result = system "openssl aes-256-cbc -d -base64 \
                 -in #{enc_backup_path.basename.to_s} -out enterprise_backup.tar \
                 -pass pass:#{password}"
{% endhighlight %}

试想如果密码中带有反引号或者美金符为怎么样？

* 额外执行三条命令来暂停回显

{% highlight ruby linenos %}
def no_echo
  _state = `stty -g`
  system 'stty -echo -icanon isig'
  yield
ensure
  system "stty #{_state}"
end
{% endhighlight %}

确定要这么复杂吗？

* 通过这样的办法在命令行中调用 LESS

{% highlight ruby linenos %}
f = Tempfile.new("help")
f.write help
f.flush
f.close
system "cat #{f.path} | less"
{% endhighlight %}

如果 UNIX 操作系统只能这么用，想必根本活不到今天了。

我们还是先来讲讲 UNIX 调用 Shell 命令的原理吧。

以下是一个 UNIX 操作系统执行 Shell 命令的 C 代码通用模版：

{% highlight c linenos %}
pid_t child_pid;
int status;

// create pipes
switch(child_pid = fork()) {
case -1:
    status = -1;
    break;
case 0:
    // process group
    // redirect / close io
    // set rlimit
    // close unused fd
    // change dir
    execl("/bin/sh", "sh", "-c", command, (char *) NULL);
    _exit(127);
default:
    // redirect / close io
    while (waitpid(child_pid, &status, 0) == -1) {
        if (errno != EINTR) {
            status = -1;
            break;
        }
    }
    break;
}
return status;
{% endhighlight %}

我们来逐句解释下这里的代码。
首先是 fork() 函数，它是 UNIX 操作系统创建新进程的经典系统调用，通过克隆自身来得到一个与自己完全一样的子进程。

{% highlight c linenos %}
pid_t fork(void);
{% endhighlight %}

一次调用，却在父子两个进程中都会返回，只是在父进程中它返回子进程的 PID，而子进程中总是返回 0。
因此，通常通过返回值来判定当前进程是父进程还是子进程。
值得一提的是，几乎所有 UNIX 系统调用都有可能出错，大部分系统调用出错时返回 -1，需要在每次调用后判定。
如果 fork() 出错，那通常是因为进程数量到达上限或者内存不足以再支撑一个进程，此时它只能返回一次，并且返回值是 -1。

waitpid() 默认情况下是一个阻塞操作，它等待指定 PID 的进程结束，然后返回进程的状态信息。

{% highlight c linenos %}
pid_t wait(int *status);
pid_t waitpid(pid_t pid, int *status, int options);
{% endhighlight %}

wait 和 waitpid 很像，只是 wait 不指定 PID，任何一个子进程结束都会导致 wait() 返回，在大型应用程序中通过 wait() 来监控子进程状态是不明智的，因为并不能确定此时返回的子进程是否与预想的一致，在多线程程序更是如此，一般总是推荐用 waitpid() 来监控指定 PID 的进程。

waitpid() 也能达成类似于 wait() 的效果，因为第一个参数 pid 还可以有其他语义：

* &lt; -1 等待任何进程组 PGID 等于 pid 的绝对值的进程
* -1      等待任何子进程
* 0       等待任何进程组 PGID 等于当前进程的 PID
* &gt; 0  等待进程 PID 等于 pid 的进程

由此可见  waitpid(-1, &status, 0) 与 wait(&status) 等价。

此外，waitpid() 还接受选项：

* WNOHANG     使 waitpid() 变成非阻塞操作，仅仅检查进程是否结束而并不等待，通过这个选项能够通过轮询监听多个进程的状态
* WUNTRACED   默认情况下，waitpid() 只在指定子进程结束后才返回，但是通过这个选项，能使 waitpid() 在子进程 Stop 后也返回
* WCONTINUED  能使 waitpid() 在已经 Stop 的子进程在恢复后也返回

需要注意的是，任何子进程都应该被父进程通过 wait() 或者 waitpid() 或是其他方法获取状态，否则子进程会变为僵尸进程，一直等待直到父进程死亡，然后由 init 进程来负责回收。

waitpid() 回收的子进程状态是一个名为 status 的 int 整型，不过它特殊的格式可以同时表示进程状态的多种可能性，例如正常返回时的返回值，被信号 Kill 或 Stop 时的信号值，以及进程是否刚刚恢复执行，可以通过宏函数来判定 status 表示的具体语义。

* WIFEXITED(status)    判定进程是否正常结束
* WIFSIGNALED(status)  判定进程是否被信号 Kill
* WIFSTOPPED(status)   判定进程是否被信号 Stop
* WIFCONTINUED(status) 判定 Stop 的进程是否被信号恢复
* WEXITSTATUS(status)  获取进程正常结束时的返回值
* WTERMSIG(status)     获取进程被信号 Kill 时的信号值
* WCOREDUMP(status)    判定进程被信号 Kill 时是否有 core dump
* WSTOPSIG(status)     获取进程被信号 Stop 时的信号值

此外，造成 waitpid() 结束的可能性还有一种就是父进程被 SIGINT 打断造成返回。此时 waitpid() 返回 -1，errno 等于 EINTR，这种情况下直接重新调用 waitpid() 即可，无需额外处理。

execl() 是 exec 函数族的一部分，所有 exec 函数族都实现一个共同的功能，执行指定的命令来取代当前进程。
这些系统调用与 fork() 刚好相反，一次调用，不再返回，除非出错。因此通常把错误处理代码直接写在 exec 调用的后面。

exec 函数族一共有下面六个成员：

{% highlight c linenos %}
int execl(const char *path, const char *arg0, ... /*, (char *)0 */);
int execle(const char *path, const char *arg0, ... /*, (char *)0, char *const envp[] */);
int execlp(const char *file, const char *arg0, ... /*, (char *)0 */);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execvP(const char *file, const char *search_path, char *const argv[]);
{% endhighlight %}

所有 exec 函数族都以 exec 开头，之后函数名中带 l 的表示将通过可变长参数来传入命令行参数，通过在最后一个参数传入 0 表示可变长参数结束，函数名中带 e 的表示末尾接受一个指针传入环境变量，函数名中带 p 表示程序需要在 $PATH 中搜索，而 execvP() 表示在指定路径下搜索程序。

一般常用的是 execl 和 execlp，其他几个函数的功能都可以通过其他手法来实现。

在上面的例子中，正是通过这段代码来执行 Shell：

{% highlight c linenos %}
execl("/bin/sh", "sh", "-c", command, (char *) NULL);
{% endhighlight %}

第一个参数表示启动的进程的路径，由于 execl 不会在 $PATH 中搜索要启动的程序，因此必须传入绝对路径。第二个参数是子进程的第 0 个参数，表示进程名。此后所有参数都是子进程的参数，直到 NULL（就是 0） 为止。

至此，模版中出现的几个系统调用已经解释完毕，但这还远远不够。UNIX 之所以将执行子进程的任务交给 fork() 和 exec() 两个系统调用组合完成，就是因为在 fork() 之后还没有执行 exec() 的子进程中，可以修改子进程的资源并影响到 exec() 之后的子进程。

在 UNIX 中，所有进程都会默认打开三个文件描述符，STDIN 表示标准输入，STDOUT 表示标准输出，STDERR 表示错误输出。

而在 fork() 时，子进程会默认共享父进程所有文件描述符。例如父进程的 STDIN 来自于键盘输入，STDOUT 将输出到 Terminal，STDERR 将输出到某个日志文件，那么子进程也会同样如此。

如果父进程想要将自己的数据输入到子进程中，或是获取子进程通过 STDOUT 或 STDERR 输出的数据，就必须通过重定向来解决。

重定向文件描述符最常见的方法是在 fork() 前先创建管道，在子进程中用 dup2() 将管道的一端重定向到某个文件描述符中，关闭另一端。在父进程中操作另一端的管道，同时关闭这一端。

下面的例子是一个父进程 fork() 子进程，并使子进程执行 ls 命令，父进程获取子进程来自于 STDOUT 的输出然后输出到自己的 STDOUT 的代码。

{% highlight c linenos %}
int pfd[2], numRead;
pid_t pid;
if (pipe(pfd) == -1) return -1;
switch (pid = fork()) {
    case -1:
        return -1;
    case 0:
        if (close(pfd[0]) == -1) return -1;
        if (pfd[1] != STDOUT_FILENO) {
            if (dup2(pfd[1], STDOUT_FILENO) == -1) return -1;
            if (close(pfd[1]) == -1) return -1;
        }
        execlp("ls", "ls", (char *) NULL);
        _exit(127);
    default:
        if (close(pfd[1]) == -1) return -1;
        for(;;) {
            numRead = read(pfd[0], buf, BUF_SIZE);
            if (numRead == -1) return -1;
            if (numRead == 0) break;
            if (write(STDOUT_FILENO, buf, numRead) != numRead) return -1;
        }
        if (close(pfd[0]) == -1) return -1;
        if (waitpid(pid, NULL, 0) == -1) return -1;
        break;
}
exit(EXIT_SUCCESS);
{% endhighlight %}

这里的 pipe() 将初始化 pfd[2]，使之成为管道。

{% highlight c linenos %}
int pipe(int fildes[2]);
{% endhighlight %}

在 UNIX 标准中，管道是单工的，pipe() 将使写入 pfd[1] 的输入可以在 pfd[0] 中被读取，仅此通常将 pfd[1] 称为写入端，而将 pfd[0] 称为读取端。
管道在单进程中没有多大意义，但是在父子进程中，利用文件描述符共享，可以实现父子进程之间的通信。

dup2() 接受两个文件描述符，然后关闭第二个描述符，随后将第一个文件描述符的内容复制到第二个文件描述符中，使之与第一个描述符一致，
达到了重定向的效果。

{% highlight c linenos %}
int dup2(int fildes, int fildes2);
{% endhighlight %}

例如在执行

{% highlight c linenos %}
dup2(pfd[1], STDOUT_FILENO);
{% endhighlight %}

之后，子进程的标准输出不再是 Terminal 而是管道的写入端，此时子进程 ls 输出的当前目录的文件列表可以在父进程中通过读取管道的读取端来获取。

子进程所做的事情还远远不止如此，它还可以关闭子进程不需要的文件描述符。

为何要关闭这些文件描述符？例如现在有父子两个进程，父进程将数据通过管道输入到子进程处理，子进程等待父进程关闭管道表示数据输入结束，然后处理数据给出结果，父进程获取子进程返回的结果后自然返回，这本来是可以正常工作的。
但如果现在父进程在 fork() 了子进程后还额外启动了一个长期执行的 Daemon 进程，并且没有在这个 Daemon 进程中关闭管道，那么 Daemon 进程同样共享了管道的写入端，当父进程关闭管道后，子进程并不会收到管道关闭时的 EOF 信息，因为管道并没有被完全关闭，因此子进程继续等待数据输入，而父进程会等待子进程结束。
除非 Daemon 进程结束，否则父子进程会形成死锁，再也无法接续执行。

关闭其他文件描述符有分聪明的方法和愚笨的方法，聪明的办法是，在每次打开一个文件描述符时，通常这些文件描述符都会接受一个叫 O_CLOEXEC 的选项，它表示当进程执行 exec 时自动关闭该描述符。你必须记住对所有文件描述符都做到这点。而愚笨的方法是这样的：

{% highlight c linenos %}
int fd, maxfd;

maxfd = sysconf(_SC_OPEN_MAX);
if (maxfd == -1)
    maxfd = 8192;
for (fd = 3; fd < maxfd; fd++)
    close(fd);
{% endhighlight %}

通过获取 sysconf(_SC_OPEN_MAX) 来确定当前进程文件描述符的最大可能值，如果获取不到就用 8192 替代，然后从 3 开始逐一调用 close() 关闭文件描述符。如果该文件描述符不存在也不要紧，因为 close() 的返回值会被忽略。
之所以从文件描述符 3 开始关闭，是因为 STDIN，STDOUT，STDERR 的编号分别是 0，1，2，一般不自动关闭这些文件描述符。

此外，还可以通过 rlimit 设置子进程占用的资源上限。UNIX 提供了这样的系统调用：

{% highlight c linenos %}
int getrlimit(int resource, struct rlimit *rlp);
int setrlimit(int resource, const struct rlimit *rlp);
{% endhighlight %}

其中 resource 相当于 key，表示被限制资源的类型，rlp 相当于 value，表示被限制的资源的阀值。

resource 支持这些 key：

* RLIMIT_AS：进程可用存储区的最大总长度
* RLIMIT_CPU：CPU 时间的最大值，当超过此软限制时，向该进程发送 SIGXCPU 信号。
* RLIMIT_DATA：数据段的最大字节长度。
* RLIMIT_FSIZE：可以创建的文件的最大字节长度。当超过此软限制时，则向该进程发送 SIGXFSZ 信号。
* RLIMIT_LOCKS：一个进程可持有的文件锁的最大数。
* RLIMIT_MEMLOCK：一个进程使用 mlock 能够锁定在存储器中的最大字节长度。
* RLIMIT_NOFILE：每个进程能打开最大文件数。
* RLIMIT_NPROC：每个实际用户 ID 可拥有的最大子进程数。
* RLIMIT_RSS：最大驻内存集的字节长度。
* RLIMIT_STACK：栈的最大字节长度。

还可以设置子进程的进程组，之前我们在讲 waitpid() 时提到了进程组。在 Terminal 中，类似于这样的命令：

{% highlight bash linenos %}
ls | wc -l
{% endhighlight %}

会生成 ls 和 wc 两个进程，它们拥有不同的进程 PID，但会拥有一样的进程组 PGID。waitpid() 就可以等待一个进程组中任何一个进程的结束，kill() 也可以向一个进程组中所有进程发送结束信号。

有时，我们并不希望子进程会和父进程一起被 Kill（例如 Daemon 进程），此时我们可以设置子进程到一个新的进程组。

{% highlight c linenos %}
pid_t setpgrp(void);
{% endhighlight %}

子进程有时需要特定的工作目录。

{% highlight c linenos %}
int chdir(const char *path);
int fchdir(int fildes);
{% endhighlight %}

有时子进程需要设置自身的环境变量，或者清空继承于父进程的环境变量。

{% highlight c linenos %}
char* getenv(const char *name);
int setenv(const char *name, const char *value, int overwrite);
int unsetenv(const char *name);
{% endhighlight %}

以上就是 UNIX 程序执行 Shell 的原理，不一定完全但对本文来说已经够用。

我们重新回到 Ruby 这个话题。

Ruby 提供了执行 Shell 命令的底层方法（其实也提供了 fork 和 exec，但本文不再讨论与 C 平级的层次）Kernel.spawn。
它实现了大量 C 中调用子进程时可能需要的功能。

{% highlight text linenos %}
Kernel.spawn([env,] command... [,options]) → pid
{% endhighlight %}

注意 Kernel.spawn 既可以接受字符串作为 Shell 命令，也可以接受数组作为参数列表。当 command 传入字符串时，将会启动 /bin/sh 对字符串解释执行。当传入数组时，讲直接调用 exec 接受参数列表。例如：

{% highlight ruby linenos %}
Kernel.spawn 'echo *'
{% endhighlight %}

会显示当前目录下所有文件，因为 * 在 Shell 中会被当前目录下所有文件名替换。而

{% highlight ruby linenos %}
Kernel.spawn 'echo', '*'
{% endhighlight %}

只会显示一个星号，因为并没有被 Shell 解释器处理过。
毫无疑问，直接被 exec 执行比起被 Shell 解释器解释执行更为高效。

Kernel.spawn 的强大功能在它的文档中完全体现。

{% highlight text linenos %}
env: hash
  name => val : set the environment variable
  name => nil : unset the environment variable
command...:
  commandline                 : command line string which is passed to the standard shell
  cmdname, arg1, ...          : command name and one or more arguments (This form does not use the shell. See below for caveats.)
  [cmdname, argv0], arg1, ... : command name, argv[0] and zero or more arguments (no shell)
options: hash
  clearing environment variables:
    :unsetenv_others => true   : clear environment variables except specified by env
    :unsetenv_others => false  : don't clear (default)
  process group:
    :pgroup => true or 0 : make a new process group
    :pgroup => pgid      : join to specified process group
    :pgroup => nil       : don't change the process group (default)
  create new process group: Windows only
    :new_pgroup => true  : the new process is the root process of a new process group
    :new_pgroup => false : don't create a new process group (default)
  resource limit: resourcename is core, cpu, data, etc.  See Process.setrlimit.
    :rlimit_resourcename => limit
    :rlimit_resourcename => [cur_limit, max_limit]
  umask:
    :umask => int
  redirection:
    key:
      FD              : single file descriptor in child process
      [FD, FD, ...]   : multiple file descriptor in child process
    value:
      FD                        : redirect to the file descriptor in parent process
      string                    : redirect to file with open(string, "r" or "w")
      [string]                  : redirect to file with open(string, File::RDONLY)
      [string, open_mode]       : redirect to file with open(string, open_mode, 0644)
      [string, open_mode, perm] : redirect to file with open(string, open_mode, perm)
      [:child, FD]              : redirect to the redirected file descriptor
      :close                    : close the file descriptor in child process
    FD is one of follows
      :in     : the file descriptor 0 which is the standard input
      :out    : the file descriptor 1 which is the standard output
      :err    : the file descriptor 2 which is the standard error
      integer : the file descriptor of specified the integer
      io      : the file descriptor specified as io.fileno
  file descriptor inheritance: close non-redirected non-standard fds (3, 4, 5, ...) or not
    :close_others => true  : don't inherit
  current directory:
    :chdir => str
{% endhighlight %}

它可以接受一个 Hash 作为第一个参数表示对环境变量的设定。
在最后的参数中传入 Hash 则代表更多语义，
如果传入的 unsetenv_others: true 表示清理其他环境变量，pgroup: true 表示创建新的进程组，
当 key 是 rlimit_[某种 resource] 时表示对子进程 rlimit 的设定。
如果 key 是 in，out，err 或者某个数字时表示 STDIN，STDOUT，STDERR 或是 数组表示的文件描述符被重定向。
而对应的 value 可以设置为 in，out，err，某个数字，某个文件路径或某个数组，数组的第一个元素表示文件路径，第二个参数表示打开模式，第三个参数表示新创建的文件的权限。
value 也可以是 :close 表示关闭该描述符。
当传入 close_others: true 时会关闭 3 以上的所有文件描述符。而传入 chdir 表示子进程当前目录切换到指定路径。

