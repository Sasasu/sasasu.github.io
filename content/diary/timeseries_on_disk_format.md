---
title: "influxdb 与 prometheus 的数据文件格式比较"
date: 2020-07-15T12:08:41+08:00
draft: false
---

## influxdb 的 time structure merge tree

time structure merge tree 现在的实现是 [tsm1](https://github.com/influxdata/influxdb/tree/master/tsdb/tsm1)

数据文件有三种：wal / tsm / tombstone

### tombstone

其中 tombstone 是墓碑，用来标识某数据是否删除。实际上时序数据库都是不支持删除数据的，这个文件结构可以忽略。这个文件的格式在 [tombstone](https://github.com/influxdata/influxdb/blob/master/tsdb/tsm1/tombstone.go)

### wal

wal 就是常规的 wal，每条数据如下

```
┌────────────────────────────────────────────────────────────────────┐
│                           WriteWALEntry                            │
├──────┬─────────┬────────┬───────┬─────────┬─────────┬───┬──────┬───┤
│ Type │ Key Len │   Key  │ Count │  Time   │  Value  │...│ Type │...│
│1 byte│ 2 bytes │ N bytes│4 bytes│ 8 bytes │ N bytes │   │1 byte│   │
└──────┴─────────┴────────┴───────┴─────────┴─────────┴───┴──────┴───┘
```

需要注意的是这是一个 entry 的格式，一个 wal 由多个 entry 组成。

influxdb 在接口上是多值的，所以 time 和 value 会不断重复。

（是的！即便是多值模型， time 在 wal 中也会写入多份）。

value 根据最前面的 type 来决定长度：对常见的 f64 来说就是结结实实的 8 byte。

在 wal 的最外面还有一层 snappy 压缩，多个 wal entry 套上一层 snappy 后形成一个 wal file。

### tsm

tsm 主要由 2 部分组成：blocks 和 index

对于 index 结构如下：

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                   Index                                    │
├─────────┬─────────┬──────┬───────┬─────────┬─────────┬────────┬────────┬───┤
│ Key Len │   Key   │ Type │ Count │Min Time │Max Time │ Offset │  Size  │...│
│ 2 bytes │ N bytes │1 byte│2 bytes│ 8 bytes │ 8 bytes │8 bytes │4 bytes │   │
└─────────┴─────────┴──────┴───────┴─────────┴─────────┴────────┴────────┴───┘
```

Index 居然是不定长的。

在 [DESIGN.md](https://github.com/influxdata/influxdb/blob/master/tsdb/tsm1/DESIGN.md#indirect-mmap-indexing) 解释如何在不定长的 Index 上检索：

首先 mmap 整个 tsm 文件（或者只 mmap index 段）然后顺序扫描一遍，在内存中构建一个变长数组来保存 index 内的 offset：

 ```
┌────────────────────────────────────────────────────────────────────┐
│                               Index                                │
├─┬──────────────────────┬──┬───────────────────────┬───┬────────────┘
│0│                      │62│                       │145│
├─┴───────┬─────────┬────┼──┴──────┬─────────┬──────┼───┴─────┬──────┐
│Key 1 Len│   Key   │... │Key 2 Len│  Key 2  │ ...  │  Key 3  │ ...  │
│ 2 bytes │ N bytes │    │ 2 bytes │ N bytes │      │ 2 bytes │      │
└─────────┴─────────┴────┴─────────┴─────────┴──────┴─────────┴──────┘
```

恩....

启动时间超长。对于每个时间需要都需要读取 （33 + tag 个数）byte 的数据。

需要占用一些内存，其内存 index 的 value 全是 u64。

第二部分是 blocks：

```
┌───────────────────────────────────────────────────────────┐
│                          Blocks                           │
├───────────────────┬───────────────────┬───────────────────┤
│      Block 1      │      Block 2      │      Block N      │
├─────────┬─────────┼─────────┬─────────┼─────────┬─────────┤
│  CRC    │  Data   │  CRC    │  Data   │  CRC    │  Data   │
│ 4 bytes │ N bytes │ 4 bytes │ N bytes │ 4 bytes │ N bytes │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
```

每个 block 都是一个时间序列的数据，每个 block data 都跟着一个 crc checksum。

Data 段开头有一个 u8 的 type 字段，标识这段 Data 的类型。

随后有一个 varint int, 其中 time 所占的长度，以便于分割 time 与 value。

然后才是具体的数据，也就说 Data 段实际上是 `[type, length, times..., values...]` 这种格式

influxdb 在接口上是多值的，但是在 blocks 层面却是单值的。在 index 层是这样拼 key 的

```go
// MIT License

func AppendMakeKey(dst []byte, name []byte, tags Tags) []byte {
	// unescape the name and then re-escape it to avoid double escaping.
	// The key should always be stored in escaped form.
	dst = append(dst, EscapeMeasurement(UnescapeMeasurement(name))...)
	dst = tags.AppendHashKey(dst)
	return dst
}

func (q *arrayCursorIterator) seriesFieldKeyBytes(name []byte, tags models.Tags, field string) []byte {
	q.key = models.AppendMakeKey(q.key[:0], name, tags)
	q.key = append(q.key, KeyFieldSeparatorBytes...)
	q.key = append(q.key, field...)
	return q.key
}
```

可见 influxdb 是一个「列存 多值」的时序数据库（笑）

有多少个字段，时间就重复了多少次。

时序数据库的正确用法是写入时间间隔相等的数据点，这样数据库做 double delta 之后再套 zstd 后时间基本不占硬盘空间。

但是时序数据库非常容易滥用，使用多值表同时写入时间间隔不同的数据点对 influxdb 来说消耗很大。

### memory

tsm 作为一个类 log structure merge tree, 还需要一个内存结构，以提供随机写入和立刻读取的能力。

tsm 中这个结构叫 [ring](https://github.com/influxdata/influxdb/blob/master/tsdb/tsm1/ring.go)

这是一个二段 hash。

第一段不扩容，固定为 16，hash 函数为无参数的 xxhash。（笑）

第二段为 go 内建的 map, key 为不定长 byte, value 是个不定长数组。

一共两组锁，第二段 hash 一组，value 内一组。

`ring` 会在内存中有两个实例，其中一个 `ring` 刷成 tsm 过程中新写入的数据会进入第二个 `ring`

`ring` 中的数据是不压缩的，按照时间戳顺序排序。

值得一提的是 `ring` 中的数据也是单值的，时间戳是不断重复的。

## prometheus 的 tsdb

Prometheus 中的时序引擎就叫 tsdb. 使用的结构也类似 LSMT

其数据文件由 chunks index wal 三部分组成

### wal

wal 中的数据根据第一个 byte 分为 3 种：Series records，Sample records，Tombstone records

Tombstone records 不重要无需关心。

Series records 为 8 byte 的块内自增 ID 加所有 tagKV 的变长字符串。

Sample records 为 8 bytes 的块内自增 ID 加 8 byte 的 timestamp，变长的 id delta，变长的 timestamp delta，8 byte 的 value。

相比 influxdb 的好处是将 tagKV 与 value 分开存可以减少一些 IO 消耗，但实际上每次 scrape 之后都会写一次 WAL，很难说能减少 IO。

恢复 WAL 时使用了一个 map 来去重，速度不会快。


### index

prometheus 的 index 包含了一个类似到排索引的结构。

分为如下几个段（废弃段略过不介绍）：

#### Symbol Table

是一个 string pool。类似 Android 中 XML resources。

存储变长字符串。后续所有的字符串均为 Symbol Table 的引用，所有引用均为 32 位。

```
┌────────────────────┬─────────────────────┐
│ len <4b>           │ num of symbols <4b> │
├────────────────────┴─────────────────────┤
│ ┌──────────────────────┬───────────────┐ │
│ │ len(str_n) <uvarint> │ str_n <bytes> │ │
│ └──────────────────────┴───────────────┘ │
└──────────────────────────────────────────┘
```

#### Series

保存所有的 TagKV 字符串，chunk 的个数与每个 chunk 的偏移量。

prometheus 在这里有个 series id 的概念，id 为此 series 在这个段中的偏移量除以 16。

其实就是偏移量。

```
One series:
┌──────────────────────────────────────────────────────────────────────────┐
│ len <uvarint>                                                            │
├──────────────────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────────────────────────────────────────────────┐ │
│ │                     labels count <uvarint64>                         │ │
│ ├──────────────────────────────────────────────────────────────────────┤ │
│ │              ┌────────────────────────────────────────────┐          │ │
│ │              │ ref(l_i.name) <uvarint32>                  │          │ │
│ │              └────────────────────────────────────────────┘          │ │
│ ├──────────────────────────────────────────────────────────────────────┤ │
│ │                     chunks count <uvarint64>                         │ │
│ ├──────────────────────────────────────────────────────────────────────┤ │
│ │              ┌────────────────────────────────────────────┐          │ │
│ │              │ c_0.mint <varint64>                        │          │ │
│ │              ├────────────────────────────────────────────┤          │ │
│ │              │ c_0.maxt - c_0.mint <uvarint64>            │          │ │
│ │              ├────────────────────────────────────────────┤          │ │
│ │              │ ref(c_0.data) <uvarint64>                  │          │ │
│ │              └────────────────────────────────────────────┘          │ │
│ └──────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Posting

存储这个 TagKV 下所有的 series id。

```
┌────────────────────┬────────────────────┐
│ len <4b>           │ num of series <4b> │
├────────────────────┴────────────────────┤
│ ┌─────────────────────────────────────┐ │
│ │ serie id <4b>  (offset in series)   │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

#### Posting Offset Table

存储这个 TagKV 到 posting 的 offset。

```
┌─────────────────────┬──────────────────────┐
│ len <4b>            │ num of entries <4b>  │
├─────────────────────┴──────────────────────┤
│ ┌──────────────────────┬─────────────────┐ │
│ │ len(name) <uvarint>  │ name <bytes>    │ │
│ ├──────────────────────┼─────────────────┤ │
│ │ len(value) <uvarint> │ value <bytes>   │ │
│ ├──────────────────────┴─────────────────┤ │
│ │ offset to posting <uvarint64>          │ │
│ └────────────────────────────────────────┘ │
└────────────────────────────────────────────┘
```

整体的思路是从 Posting Offset Table 出发，找到许多 Posting Offset，再找到许多 Series 再找到许多 Chunk Offset, 选择一堆 Cunk 读取。

但是 Posting Offset Table 是不定长的，没法进行二分搜索。

想必一定是在启动的是时候扫描 Posting Offset Table 构建一个可以二分搜索的结构。

内存开销不会小，启动时间不会短。并且字符串存了两次。

#### Chunk

时序数据的实际文件。

```
┌───────────────┬───────────────────┬──────────────┬────────────────┐
│ len <uvarint> │ encoding <1 byte> │ data <bytes> │ CRC32 <4 byte> │
└───────────────┴───────────────────┴──────────────┴────────────────┘
```

其 data 是 time 与 value 交替存储，time 做 double delta，value 做 teller 编码。

这地方也比较奇怪，time 一般来说是间隔相等的，所以会是一堆 0b。

time 与 value 交替存储会导致压缩率较低。

#### memory

作为一个 LSMT，prometheus tsdb 也有内存部分，叫做 [head](https://github.com/prometheus/prometheus/blob/master/tsdb/head.go)

这个结构使用一个 chunk 数组来作为内存索引，series id 使用递增变量而不是 offset。

内存索引使用一个二段数组，第一段大小固定为 16384，第二段不定长。key 为 series tagKV 的 xxhash。value 为带状态的写入器。

恩... 一个简单地 hash table.

这个写入器直接以 on disk chunk 格式保存，带压缩。

搜索部分结构如下 [posting](https://github.com/prometheus/prometheus/blob/05038b48bdf08dd72731bfaf74c9f5e5eaff5f39/tsdb/index/postings.go)

```go
// Copyright 2017 The Prometheus Authors
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
// http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

type MemPostings struct {
	mtx     sync.RWMutex
	m       map[string]map[string][]uint64
	ordered bool
}

func (p *MemPostings) Get(name, value string) Postings {
	var lp []uint64
	p.mtx.RLock()
	l := p.m[name]
	if l != nil {
		lp = l[value]
	}
	p.mtx.RUnlock()

	if lp == nil {
		return EmptyPostings()
	}
	return newListPostings(lp...)
}

func (p *MemPostings) addFor(id uint64, l labels.Label) {
	nm, ok := p.m[l.Name]
	if !ok {
		nm = map[string][]uint64{}
		p.m[l.Name] = nm
	}
	list := append(nm[l.Value], id)
	nm[l.Value] = list

	if !p.ordered {
		return
	}
	// There is no guarantee that no higher ID was inserted before as they may
	// be generated independently before adding them to postings.
	// We repair order violations on insert. The invariant is that the first n-1
	// items in the list are already sorted.
	for i := len(list) - 1; i >= 1; i-- {
		if list[i] >= list[i-1] {
			break
		}
		list[i], list[i-1] = list[i-1], list[i]
	}
}
```

恩...

行吧，能用。

