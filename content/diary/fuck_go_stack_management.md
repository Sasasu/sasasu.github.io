---
title: "写出无入侵的链路追踪 go sdk 是不可能的"
date: 2019-07-14T17:31:08+08:00
draft: true
---

# 目标

现在，在你的面前有一坨二进制和这一坨符号表。现在你想获得二进制中某个库函数的运行时间、以及函数调用时传入和返回的参数，最后还需要关联两个进程之间的线程。

我们从一个应用程序的生命周期开始入手，找出一个合适的方式植入一个链路追踪 sdk。

# 程序编译时

大多数网络程序都会使用某种网络库，只需要在编译或者链接时将 lib 替换掉，收到 rpc 时将调用特征保存进 thread local，发送 rpc 时继续将 thread local 传递即可。

在 c/c++ 的世界里只需要改掉编译系统的 lib 即可。对于 c++ 模板来说需要在编译前修改，有点难以推行但不是问题、对于 c 来说只需要一个 `LD_PRELOAD` 环境变量替换掉运行库即可。

但是大多数 go 程序使用 vendor 管理依赖，将依赖作为自己代码的一部分，在不修改 go 编译器和用户代码的情况下没法修改运行库。编译时进行修改的计划失败了。

# ld 载入二进制时

操作系统在运行程序前还有一步解决符号重定位的步骤。但是 Go 使用静态链接，不进行这一步。一棒子打死。

## 修改二进制

万幸，Go 有选项可以强制使用动态链接，尽管它并没有动态链接任何东西。所以给了我们使用 `__constructor__` 来在 Go 启动前修改内存中的二进制的能力。

修改二进制有多种方法，可以去读一下 [x86 API Hooking Demystified](http://jbremer.org/x86-api-hooking-demystified/) [^1]。

因为我想获取和传递调用特征，所以替换运行时的网络库是最方便的。

在 x86 平台中 Go 使用 `call` 来调用一个函数，使用 `ret` 从函数中返回。这符合 X86 CPU 的工作方式，Go 没作出什么反直觉的改变。所以我想直接在函数头部放一个 jmp 来跳转到新的库函数上。

当然也有其他的方法，比如以可以塞一个 int3 在某个地方，然后在内核里或者进 debuger 里处理，也可以魔改二进制跳来跳去。直接一个 long jump 把函数换掉或许是最简单的。

要把函数换掉首先要知道函数的地址，使用 dlopen 然后 dlsym 就可以获取函数的地址，然而不幸的是 Go 的所有函数均没有导出，没法用 libc 提供的东西了。

Linux 系统中使用 dlopen 实际上是用 mmap 把文件装进内存并用 memprotect 设置一些属性。即便 lib 编译时使用了 pie ，在 linking 之后整个内存依旧是连续的。各个地址的相对值不会发生变化。

所以我们首先需要手动从符号表中分析出某个函数的首地址偏移量，在 dlopen 之后再计算出 .so 的基地址，再算出函数的开始地址，将函数开头塞入一个 jmp 并跳入一个新的 Go 函数，干该干的事情最后用 ret 返回。

x86 使用变长指令集，实际上在 64 位中并没有能直接跳 64 位的指令。所以 long jump 实际上是一个 push + ret，一共需要 6 个 byte。

几乎所有的 Go 函数开头都是 `mov + cmp + jbe`，然后去检测栈是否需要扩容，放入 6 byte 的代码没有任何问题，使用 4byte 的栈一般没啥问题。但实际上还是我栽在了编译器插入的这段代码里。

在查阅 go abi 规范后得知：Go 不使用任何寄存器传递参数，参数全靠栈。可以放心大胆的使用一个寄存器。

最后实现的代码长这样：

https://gist.github.com/Sasasu/46175f37bd233e72c81636583809cdba

## 栽了

然而结局想当无情，Go 崩溃了

```
[Step 1] finding Go's http/net.Get
self dl base 0x55fb87964000
self func printer is 0x55fb87c838c0
[Step 2] finding hook's http/net.Get
hook dl base 0x7fcf3ed23000
hook func printer is 0x7fcf3f260c50
[Step 3] open memeory
self memeory page base is 0x55fb87c83000
[Step 4] inject the code
jump from 0x55fb87c838c0 to 0x7fcf3f260c50, offset = 0xb75dd38b inject done.
0:  48 b8 00 00 7f
5:  cf 3f 26 0c 50
10: 50 c3
Go Start
unexpected fault address 0x0
fatal error: fault
[signal SIGSEGV: segmentation violation code=0x80 addr=0x0 pc=0x55b5fa2d38cb]

goroutine 1 [running]:
runtime.throw(0x55b5fa31a553, 0x5)
	/usr/lib/go/src/runtime/panic.go:617 +0x74 fp=0xc0000bbec8 sp=0xc0000bbe98 pc=0x55b5fa114d04
runtime.sigpanic()
	/usr/lib/go/src/runtime/signal_unix.go:397 +0x405 fp=0xc0000bbef8 sp=0xc0000bbec8 pc=0x55b5fa129355
runtime: unexpected return pc for net/http.Get called from 0x508c67ba0a7f0000
stack: frame={sp:0xc0000bbef8, fp:0xc0000bbf00} stack=[0xc0000ba000,0xc0000bc000)
000000c0000bbdf8:  0000000000000001  0000000000000000
000000c0000bbe08:  000000c0000bbe48  000055b5fa115886 <runtime.gwrite+166>
000000c0000bbe18:  0000000000000002  000055b5fa31a044
000000c0000bbe28:  0000000000000001  000055b500000001
000000c0000bbe38:  000000c0000bbeb5  0000000000000003
000000c0000bbe48:  000000c0000bbe98  000055b5fa1160ac <runtime.printstring+124>
000000c0000bbe58:  000055b5fa114ed9 <runtime.fatalthrow+89>  000000c0000bbe68
000000c0000bbe68:  000055b5fa13d390 <runtime.fatalthrow.func1+0>  000000c000000180
000000c0000bbe78:  000055b5fa114d04 <runtime.throw+116>  000000c0000bbe98
000000c0000bbe88:  000000c0000bbeb8  000055b5fa114d04 <runtime.throw+116>
000000c0000bbe98:  000000c0000bbea0  000055b5fa13d300 <runtime.throw.func1+0>
000000c0000bbea8:  000055b5fa31a553  0000000000000005
000000c0000bbeb8:  000000c0000bbee8  000055b5fa129355 <runtime.sigpanic+1029>
000000c0000bbec8:  000055b5fa31a553  0000000000000005
000000c0000bbed8:  0000000000000001  0000000000000000
000000c0000bbee8:  000000c0000bbf88  000055b5fa2d38cb <net/http.Get+11>
000000c0000bbef8: <508c67ba0a7f0000 >000055b5fa318e8c <main.main+172>
000000c0000bbf08:  000055b5fa31c5b3  000000000000000e
000000c0000bbf18:  0000000000000001  0000000000000009
000000c0000bbf28:  0000000000000000  0000000000000000
000000c0000bbf38:  0000000000000000  000000c0000bbf48
000000c0000bbf48:  000055b5fa3fa020  000055b5fa45b350
000000c0000bbf58:  000000c000082058  0000000000000000
000000c0000bbf68:  0000000000000000  000000c0000bbf48
000000c0000bbf78:  0000000000000001  0000000000000001
000000c0000bbf88:  000000c0000bbf90  000055b5fa1166a4 <runtime.main+532>
000000c0000bbf98:  000000c000082000  0000000000000000
000000c0000bbfa8:  000000c000000000  0000000000000000
000000c0000bbfb8:  0000000000000000  0000000000000000
000000c0000bbfc8:  000000c000000180  0000000000000000
000000c0000bbfd8:  000055b5fa1408e1 <runtime.goexit+1>  0000000000000000
000000c0000bbfe8:  0000000000000000  0000000000000000
000000c0000bbff8:  0000000000000000
net/http.Get(0x55b5fa318e8c, 0x55b5fa31c5b3, 0xe, 0x1, 0x9)
	/usr/lib/go/src/net/http/client.go:369 +0xb fp=0xc0000bbf00 sp=0xc0000bbef8 pc=0x55b5fa2d38cb
```

报错的原因和那句 `unexpected fault address 0x0` 无关，主要原因在于 `unexpected return pc for net/http.Get called from 0x508c67ba0a7f0000`。

撇一眼地址，似乎没跳过去，64 位下 near return 会弹出 8 byte，指令为 c3。没有任何问题... 在目标函数头上放一个 int3 也能证明确实跳过去了。

于是现在代码在那里 panic 了完全没有头绪，只好去 issues 中找一找：

https://github.com/golang/go/issues/23133

这个 issues 是关于 plugin 在 mac os 上 panic 的，但可以参考。

Go 为了生成调用回溯，Go 会检查调用者的指针，在 runtime 里找不到对应的函数名时便会 panic。

我们死在了第二个 runtime 的入口。那句 unexpected return pc 是新 runtime 打出来的。

在对 Go api 有所了解之后发现 Go 的接收者，多返回值均为语法糖。

`func (*Sa) F(int) (int, int)` 会变成 `func F(*Sa, int, int, int)` 原来多返回值是真的多返回值，而不是引入 Tuple 类型。

## 尾声

最后还是在原地放一个 int3，打个断点去 lldb 吧。

# Google 怎么做的？

给程序做链路追踪应该是一个常见需求，为此我拜读了世界上第一篇链路追踪的论文 Dapper。

这篇论文中有这么一句话：

> In our environment, since all applications use thesame threading model, control flow and RPC system

线程模型，控制流？？

哦。这就大概是 Go。

# appDynamics 是怎么做的？

大概是打了个断点进去。

[^1]: X86 API Hooking Demystified http://jbremer.org/x86-api-hooking-demystified/
