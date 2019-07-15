---
title: "无入侵地获取 go 函数的执行时间"
date: 2019-07-14T17:31:08+08:00
draft: true
---

如果你正在寻找一个完美的解决方案，直接放弃。因为 Go 不遵守 C 的函数调用约定，而外部无法得知 Go runtime 的运行状态。

这篇文章类似一个导读，会依次介绍现在常用的 trace 方法，并给出对应的阅读链接。但是不会详细说明具体要如何编写代码来实现这种功能。

# 目标

现在，在你的面前有一坨二进制和这一坨符号表。现在你想获得二进制中某个函数的运行时间、最好还有函数调用时传入和返回的参数。

# ld 载入二进制时修改

既然 Go 使用纯静态链接，那我们直接改掉二进制。

# 修改二进制

在 x86 平台中 Go 使用 `call` 来调用一个函数，使用 `ret` 从函数中返回。这符合 X86 CPU 的工作方式，Go 并没有把调用者存在奇怪的地方然后用 `jmp` 来调用函数和返回调用者，很符合直觉（毕竟 call 比 push + jmp 更快）。于是，理论上可以使用 C/C++ 的方式来获取函数的执行时间。

具体的原理可以去阅读 [x86 API Hooking Demystified](http://jbremer.org/x86-api-hooking-demystified/) [^1] 这里介绍一下最基本的方式。

首先需要从符号表中分析出某个函数的首地址，然后把这个首地址替换成 `jmp` 指令，让控制流程进入自己写的函数中，保存寄存器，获取开始时间，恢复寄存器并跳转回原函数。对于 `ret` 指令也如法炮制，把它换成 `jmp` 干完事再调回来。

但是 x86 使用变长指令集，并且实际上有两大种 jump 指令 [^2]，一种称为 long jump，另一种称为 short jump。考虑一种相对简单地 `JMP rel32` 它使用 1 byte 的指令和 4 byte 的 32 位相对地址。

一个 `jmp` 需要花费 5 个 byte ！所以你需要手动甚至人肉解析 x86 汇编代码，在函数头部空出 5 byte，把这 5 byte 放进你自己的函数中，在原来的函数位置用 1 byte 的 `nop` 对齐。

最后函数头部看起来会是这样

![](/img/fuck_go_stack_management/ah-basic-api-hook.png)

万幸 Go 程序一般来说一上来就是一个 `mov + cmp + jbe`，形式十分固定，空间足够塞入一个 `jmp`。同时把插入函数的使用的栈空间压缩 10 byte 之内，实在不行栈用完跑崩了就装傻让用户重启 XD

在 C++ 的世界中 GCC 和 Clang 都有一个 `patchable-function-entry` 的编译选项[^3]，会在函数头部加入无用代码填充空间。
但是它们有 `finstrument-functions` 可以自动调用前和返回后插入函数调用[^4]...

那么对于 C/C++ 来说改改编译脚本就好了 = = 不需要这么麻烦，为何要自找无趣。

函数头部勉强可以解决，但是函数尾部就没办法了。`ret` 指令只有 1 byte，毫无 hook 方法。别人可能一个 `jmp` 就跳到 `ret` 头上了，不可能在 `ret` 前面加 `jmp` 的。

等等，`int3` debug 中断只有 1 byte，可以 jump 到指定地址。有希望。

# uprobes、uretprobes

systemtap、bcc、ebpf、pftrace

# 线程、Goroutine、栈扩容、多返回值

# 断点

# Google 怎么做的？

线程库，控制流？？

哦。这就是 Go


[^1]: X86 API Hooking Demystified http://jbremer.org/x86-api-hooking-demystified/
[^2]: X86 jmp reference https://www.felixcloutier.com/x86/jmp
[^3]: patchable-function-entry https://developers.redhat.com/blog/2018/03/26/gnu-toolchain-update-2018/
[^4]: intel 关于 finstrument-functions 的介绍 https://software.intel.com/en-us/cpp-compiler-developer-guide-and-reference-finstrument-functions-qinstrument-functions
