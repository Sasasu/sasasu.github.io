---
title: "VictoriaMetric 数据文件格式"
date: 2020-07-17T11:47:48+08:00
draft: false
---

VictoriaMetric 的数据文件基本仿照 ClickHouse 的格式。

去除掉允许改表的结构，加一层 TSID 的索引为时序特性特化。

类似于有两个字段的表：timestamp 和 values。
  - timestamp 基本等价 `timestamp BigInteger Codec(DoubleDelta)`
  - values 基本等价    `values Float64 Codec(Gorilla)`

数据由 timestamp.bin / values.bin / index.bin / metaindex.bin 组成。

#### TSID

即唯一 ID，ID 由 MetricGroup / JobID / InstanceID / MetricID 组成。

即 prometheus 中固定存在的 `__name__`, `job`, `instance` 三个 tag 字符串做 xxhash64 截断拿到 id 上。

id 为定长 24byte。

MetricGroup / JobID / InstanceID 三个 tag 有明显的层级结构。

MetricID 为自增的原子变量，起点为服务启动时的纳秒值。

```
TSID: 24 byte
  MetricGroup: 8 byte
  JobIB:       4 byte
  InstanceID:  4 byte
  MetricID:    8 byte
```

#### Part

一个 part 由 timestamp.bin / values.bin / index.bin / metaindex.bin 四个文件组成。

part 是一个内存结构。一个 part 代表一组文件。

##### Part Header

part 的 header 写在文件夹名上，各个分段间由 `_` 分割。

```
Part Header:    string
  RowsCount:    string
  BlocksCount:  string
  MinTimestamp: string
  MaxTimestamp: string
```

当装载一个 Part 时 `.metaindex` 这个文件需要读入内存。

#### Meta Index

```
Meta Index Row: 56 byte
  TSID:              24 byte
  BlockHeadersCount: 4  byte
  MinTimestamp:      8  byte
  MaxTimestamp:      8  byte
  IndexBlcokOffset:  8  byte
  IndexBlcokSize:    4  byte
```

`metaindex.bin` 由多个 Meta Index Row 使用 zstd 压缩后组成。

在内存中解压后 meta index 可以进行二分搜索，通过 TSID 定位 index block offset。

Meta Index Row 使用了结构体，为了两个 4 byte 的字段需要做对齐，实际占用 64 byte 保存一个 row。

#### Index Block

`index.bin` 中存储连续存放的 Block Headers。

从 Meta Index 拿到 offset 后可以在这里找到对应的 Block Header 数组。

将属于同一个 TSID 的 Block Header 连续存放，外面套上 zstd 常规压缩后在连续存放。

即为 index.bin。

```
Block Header: 81 byte
  TSID:                 24 byte
  MinTimestamp:         8  byte
  MaxTimestamp:         8  byte
  FirstValue:           8  byte
  TimestampBlockOffset: 8  byte
  ValueBlockOffset:     8  byte
  TimestampBlockSize:   4  byte
  ValueBlockSize:       4  byte
  RowsCount:            4  byte
  Scale:                2  byte
  TimestampMarshalType: 1  byte
  ValueMarshalType:     1  byte
  PrecisionBits:        1  byte
```

其中 Scale 为 value 的倍数。存储的数据 = (int)真实数据 * Scale

PrecisionBits 是当使用 NearestDelta2 这种编码时的有效位长度。

拿到 Block Header 后即可定位到多组 TimestampBlockSize 和 ValueBlockOffset，分别去 `timestamp.bin` 和 `values.bin` 中寻找数据。

#### Block

TimestampBlock 与 ValueBlock 文件结构上无区别。文件中只有数据。

比较神奇的是 VictoriaMetric 没有用 XOR 编码。

https://medium.com/faun/victoriametrics-achieving-better-compression-for-time-series-data-than-gorilla-317bc1f95932

首先将所有数据 * Scale 转化为整数。然后数出最长的公共的后缀 0 的个数，再做除法。

注意这都是 10 进制的。

然后有如下 4 种编码方式：

```go
// constantly changed time series with constant delta. 固定间隔的值
MarshalTypeDeltaConst = MarshalType(2)

// time series containing only a single constant. 全一样的值
MarshalTypeConst = MarshalType(3)

// counter timeseries. 即所谓的 double delta
MarshalTypeNearestDelta2 = MarshalType(5)

// gauge timeseries. 只取一次 delta
MarshalTypeNearestDelta = MarshalType(6)
```

在这 4 中编码方式上再加一层 ZSTD 就有了另外 4 中压缩方式。

#### in memory part

ClickHouse 没有 in memory part, 要求客户端为输入做排序，并大块插入。

VictoriaMetric 也有类似的设计。在 agent 中处理不连续的数据，转换成 promethrus remote write 协议向存储发送。

在数据发送前是不可查询的。

#### index
名为 merge set, 结构依旧类似 ClickHouse。

由 `metaindex.bin` `index.bin` `items.bin` `lens.bin` 这四种文件组成

merge set 是一个前缀 kv。使用 Seek 实现倒排索引。

其时序部分依旧在 [sotrage/index_db](https://github.com/VictoriaMetrics/VictoriaMetrics/blob/master/lib/storage/index_db.go) 中实现。

索引有 6 个

```
0 + MetricName(字符串) + TagKV(有序 字符串数组)               -> TSID

1 + MetricName(字符串)                                        -> MetricID
1 + TagK(字符串) + TagV(字符串)                               -> MetricID

2 + MetricID(8 byte)                                          -> TSID(24 byte)

3 + MetricID(8 byte)                                          -> MetricName(字符串) + TagKV(有序 字符串数组)

4 + MetricID(8 byte)                                          -> 空，标识已经删除的 metric

5 + 时间(8byte)                                               -> MetricID(8byte)

6 + 时间(8byte) + MetricName(字符串) + TagKV(有序 字符串数组) -> MetricID
```

