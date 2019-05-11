---
title: "对静态连接也有效的伪装终端"
date: 2019-05-11T13:20:02+08:00
draft: false
---
许多程序在输出日志时会区分目的地是不是一个终端，如果目标是终端则输出人类友好的日志格式，如果不是终端则输出机器友好的日志格式。
同样，如果某程序设计在终端中使用，将文件重定向到 stdin 时程序会显示警告，或者直接罢工。

但是，有时候我们想使用管道来同时查看和保存日志的输出。类似 `ls | tee log.save`，此时这些程序就一点也不友好了。

这些大多数程序使用 `libc` 内置的 `isatty` 函数来判断某个文件描述符是不是一个终端。
在理想情况下简单地使用 `LD_PRELOAD` 覆盖掉 `isatty` 函数就可以了。
像是 [stdoutisatty](https://blog.lilydjwg.me/2013/7/9/pretend-that-stdout-is-a-tty.39922.html) 就是这样子实现的。
这样子兼容性最好，性能也最好。

少数程序会判断文件描述符号的缓冲模式，来判断是否是一个终端。
如果文件描述符是行缓冲，则输出机器友好的格式；如果描述符是无缓冲，则使用人类友好的格式并更快速地打印日志。
这种情况下在 shell 对新的可执行文件执行 exec 前调整终端或文件的缓冲模式即可。

最麻烦的是静态连接 libc 或者直接进行系统调用的程序。

全静态连接的软件不太常见，现在最常见的应该是某语言写的命令行工具。

在这种类型的语言生态中完全由自己完成系统调用，不触碰 libc。
甚至有人重写了 libc 中的 `isatty` 函数，使其更方便的直接进行系统调用。
比如更新活跃的 [matten/go-isatty](https://github.com/mattn/go-isatty)。

libc 中的 `isatty` 使用 `tcgetattr` 获取终端属性，如果成功则认为此文件描述符是一个终端。`tcgetattr` 最终会使用 `ioctl` 系统调用，获取终端属性。

go-isatty 直接向 `ioctl` 传入 `TIOCGETA` 获取终端属性。它直接进行 `syscall`，我们在非特权下没有任何 hook 的余地。

万幸，Linux 中有一个无须特权就可以使用的伪终端驱动，它对终端属性有关的系统调用全部返回成功，不设置任何东西。

所以我写了一个程序。创建伪终端，并强制子程序认为自己在终端输出。

https://github.com/Sasasu/ColorThis

使用帮助：
```bash
Usage: ColorThis <option> [application]
where options are
        -stdin:   make stdin is a tty
        -stdout:  make stdout is a tty
        -stderr:  make stderr is a tty
        -hook:    hook libc's isatty function
                  NOTE: may not work well
where application like
                  ls / </dev/null 2>/dev/null
```

对于如果输入 `-stdin` `-stdout` `-stderr` 当中的任意一个，会创建一个对应的伪终端，并将伪终端向子进程传递。

同时，依然保留了前面介绍的使用 `LD_PRELOAD` 来覆盖 `isatty` 的方式。毕竟这种方式兼容性和性能都是最好的。

程序使用了 epoll，所以不兼容 Mac OS 和其他 BSD 系统。

这之后就可以帅气的运行

```
ColorThis -stdin -stdout -stderr $SOME_FRIENDLLY_GO_APP $COMMAND 2> colorfull_err_log.log < input_from_terminal.txt | tee colorfull_log.log
```

来同时做到保存彩色 log、复用输出流、脚本化输入。甚至，你可以在这里面打开 vim 这种级别的 tui 程序。

\w/
