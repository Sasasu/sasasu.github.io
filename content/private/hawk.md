---
title: "Hawk"
date: 2018-06-28T22:58:46+08:00
draft: fasle
---

[PDF](/pdf/atc15-paper-delgado_update.pdf)

# 目标

1. 吞吐量
2. 延迟
3. 资源利用率

QoS

# 在 Sparrow 之前

一个 job 下分好多个 Task, 都交给一个中心调度器调度.

中心调度器了解一切, 调度开销大.

利于: 利用率, 吞吐量
不利: 延迟

# Hawk 的前辈 Sparrow

调度器分成了多个, Job 随机选择调度器调度.

1. Sparrow 调度器随机选择两个 Worker 发送 per-task, 获取 Worker 的等待队列长度
2. 选小的那个发送 Job

利于: 延迟, 异构计算
不利: 大 job 小 job 混搭会有巨大延迟, 调度器监控几个 Worker 不好调参

# Hawk

大任务走中心调度器, 小任务走分布式调度器

1. Sparrow 的方法
2. 偷工作, Work 去主动把其他 Work 的等待队列里的活倒序拿过来
3. 资源保留, 一小部分资源只留给短作业 (10-20%)

# 总结

都是一个组做的, 基本上只适合 Spark SQL 这样的小作业

