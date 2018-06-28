---
title: "Resource Central SOSP17"
date: 2018-06-28T10:24:24+08:00
draft: true
---

[PDF](/pdf/Resource-Central-SOSP17.pdf)

[数据集 120G](https://github.com/Azure/AzurePublicDataset)

介绍了 Azure 中的虚拟机使用情况, 用帐号的偏好做了个决策树

# 类型
IaaS 和 PaaS
数量比为 47:53
占用资源比为 23:77

PaaS按使用量计费, 买使用量比买设备便宜.

96%的账户只会创建一种虚拟机, 用IaaS的不用 PaaS (why?

# 利用率

基本在 45%-60%, 第三方利用率高于第一方(不要钱随便用就会浪费系列), 25%的第三方虚拟机利用率极高(挖矿的吧)

60%的人会选择 1c2g 的配置, 普遍偏好小机器

# 规模

90%的账户同时创建的虚拟机数小于15

# 生存时间

90% 的虚拟机生存时间小于24小时, 50%的小于1小时. 超过一天的虚拟机就会活很久, 生存时间与资源利用率没有强关联

# 复杂分类

FT找周期, 有周期的是延迟敏感, 没有的是延迟不敏感, 有90%的正确率

# 相关性

![](/img/resource-centra-usage/corr)

