---
title: "如何滥用 protobuf"
date: 2018-11-07T22:44:35+08:00
draft: fasle
---

从几年前开始, 一个流行的趋势是 web 框架自带序列化反序列化, 同时也不内嵌某种模板语言, 而是直接输出 json, 甚至有一些专做这个的框架, 比如 [serde](https://github.com/serde-rs/json). 这样不仅方便 app 们用 native 的代码去解析后端给的数据, 同时也满足了前端工程化的愿望.

这些框架大概是这么用的

```rust
// rocket (rust)
#[derive(Deserialize)]
struct Task {
    description: String,
    complete: bool
}

#[post("/todo", data = "<task>")]
fn new(task: Json<Task>) -> String { ... }
```

```java
// spring (java)
@Data
class Task {
    public String description;
    public bool complete;
}
@Controller
public class FuckJavaController {
    @PostMapping("/todo")
    public String new(@RequestParam(name="task", required=true) Task task) { ... }
}
```

看起来很美好, 调用很干净. 但是使用 api 的肯定不止一种语言, 使用某种语言定义的结构另一种语言并不能直接使用, 对外服务一般会有专门的sdk, 对内服务则是谁用谁解析. 于是已经用代码结构化的数据又被重新定义了一边, app 们每次做新功能都要先等后端加字段, 甚至会多线程请求详情接口, qps爆炸.

解决这个问题需要有一种中立, 简单, 无逻辑的语言来定义数据结构, 并且这种语言可以编译成其他的语言. protobuf 不仅是一种传输协议, 它还有一个 DSL, 刚好可以用来做这件事.

所以我想滥用一下 protobuf, 只用它定义结构, 不是用传输协议.

protobuf 的编译器叫做 protoc, 它从参数总找输入的 .proto 文件, 向插件的 stdin 输入编译好的proto文件, 从插件的 stdout 读输出的代码, 然后写入对映的文件. protobuf 有两种 syntax, 协议层互相兼容. 3比2少了很多东西, 但结构完全相同, 只支持 syntax3 的话就是少处理几种 option 而已. 里面还有一点疑似 protobuf1 的东西, 感觉整个 protobuf 应该是从xml上改过来的.


上面的代码用 protobuf 写就是这种样子

```proto
syntax = "proto3";

message Task {
    string description;
    bool comlete;
}

message EmptyMessage {}

service FuckProtocService{
    rpc new(Task) returns (EmptyMessage){
        (foo.bar.http) = {
            post: "/todo"
        }
    }
}
```

然后运行 protoc 调用插件就可以生成对映的其他语言代码.

传输的数据要求可读性时, 可以序列化成 json; 要求速度时可以直接用 protobuf; 最重要的是这一切对下层透明(虽说为了支持 oneof any 会带一个runtime)

实现还是有点坑, 我在这里列举一下

1. json 类型太少了, http 东西太多了
    - 简单地水桶原理
    - 为了兼容 json, protobuf 定义的几种类型都会被变成 number 类型
    - 反序列化时只能尽力还原原本的信息 (大于64位的整数, 浮点数输出json的精度)
    - json 没有二进制, 传输层协议会变
1. protoc 没有做 message 之间的依赖分析
    - 花10分钟写, 2小时debug
1. 看到很多 **only for google internal usage** 很不爽
