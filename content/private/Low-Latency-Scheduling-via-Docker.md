---
title: "Low Latency Scheduling via Docker"
date: 2018-06-28T22:53:30+08:00
draft: true
---

[PDF](/pdf/atc17-chen_wei.pdf)
[slide](/pdf/atc_slides_chen_wei_0.pdf)

# 现象

1. 过长的作业占用了大量资源, 短作业没法调度
2. 普遍实现是杀掉长作业空出资源
3. 主流的框架不支持抢占


 框架 | 年份 | 做法 | 缺点
 -----|-----|-----|-----
YARN|13年|杀了长作业|overhead太大
Sparrow|13|per-task|没有作者想要的抢占
Hawk|15|保留一部分机器|很难预计保留多少
Natjam|13|checkpoint|存档的频率
Amoeba|12|checkpoint|存档的频率
CRIU|15|按序存档|入侵性太强
Borg|15|任务隔离|还是killbase


kill-base 实现在 spark 中大概 80% 左右的overhead, mapreduce 中对于 map-heavy 是10% 对reduce-heavy 是50%.

map-heavy 的作业重启比较方便, 不会有多大的开销.

# 做法

1. 在 docker 中启动任务
2. 必要时压缩资源 (明抢啦

