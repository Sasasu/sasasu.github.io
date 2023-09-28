---
title: "X86 SIMD 指令小抄"
date: 2023-09-20T11:30:15+08:00
draft: true
---

[Intel 手册](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)的快速检索版本


# MMX

与浮点数寄存器（stX）共用，宽度 80bit，数量 8 个 (MM0 - MM7)。
编译器参数 -mmmx。

```
padd: L1T0.5 R = A + B

<b> 8 个  8bit 有符号整数
<w> 4 个 16bit 有符号整数
<d> 2 个 32bit 有符号整数
<u> 无符号整数
<s> 饱和运算 L1T1
```

饱和运算：不会溢出，会保持在最大值。
https://zh.wikipedia.org/wiki/%E9%A5%B1%E5%92%8C%E8%BF%90%E7%AE%97

```
pmaddwd: L4T1 (A'32, B'32) = (A1'16*B1'16), (A2'16*B2'16)
pmulhw:  L5T1 R'32 = A'16 * B'16[高16bit]
pmullw:  L5T1 R'32 = A'16 * B'16[低16bit]
```

```
pand:  L1T0.5 R = A AND B
pandn: L1T0.5 R = (NOT A) AND B
por:   L1T0.5 R = A OR B
```


```
pcmpeq: L1T1 R = (A == B) ? 0xFFFF : 0
pcmpgt: L1T1 R = (A >  B) ? 0xFFFF : 0

<b> 8 个  8bit 有符号整数
<w> 4 个 16bit 有符号整数
<d> 2 个 32bit 有符号整数
```


```
mov: A = B

<q> 64bit L2T1
<d> 32bit L1T1
```


```
emms: reset A

L10T4.5
```

```
packssw: L3T2 复制 A1...An = B

<b> 8bit
<w> 16bit
<u> 无符号
```


# SSE

独立寄存器，宽度 128bit，32bit 下数量 8 个，64bit 下数量 16 个。额外一个控制寄存器 MXCSR
XMM{0,16}


