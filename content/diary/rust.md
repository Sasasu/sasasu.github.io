---
title: "Rust"
date: 2018-08-28T22:13:27+08:00
draft: true
---

## 变量声明

默认变量都是 const
let a = 1
let mut a = 1

&是取地址符翻译成借用 brow

匹配模式

match Enum{
    A::A => println!(""),
    A::B => println!(""),
    _ => println!("")
}

## 错误处理

Result 类型
Result 是个枚举 里面有 Ok 和 Err

is_ok is_err
ok() -> Option
err() -> Option
