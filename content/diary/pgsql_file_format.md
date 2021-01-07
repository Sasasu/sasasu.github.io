---
title: "PostgreSQL 的文件格式"
date: 2020-12-31T13:25:12+08:00
draft: true
---

[doc](https://www.postgresql.org/docs/13/storage-file-layout.html)

PostgreSQL 使用 "共享硬盘" 和 "共享内存" 的方式在单机上组成集群。尽管文档不断重复 "Cluster"，PostgreSQL 只能单机运行。

从文件层面 PostgreSQL 表现如下：

| 文件名                            | 作用                                   |
| -----------                       | -----------                            |
| [PG_VERSION](PG_VERSION)          | 版本号                                 |
| [base/NNN/PG_VERSION](PG_VERSION) | 版本号                                 |
| [base/NNN/MMM](PageFile)          | NNN 库 MMM 表数据                      |
| [base/NNN/MMM_vm](VisibilityMap)  | NNN 库 MMM 表可见块                    |
| [base/NNN/MMM_fsm](FreeSapceMap)  | NNN 库 MMM 表空闲块                    |
| [base/NNN/MMM_init](InitFile)     | NNN 库 MMM 表初始化标记                |
| current_logfiles                  | 日志文件当前写入                       |
| global                            | pg_global 库里的表类似 base/NNN        |
| pg_commit_ts                      | 事务时间戳                             |
| pg_dynshmem                       | 动态共享内存                           |
| pg_multixact                      | 共享行锁                               |
| pg_notify                         | pg_notify                              |
| pg_logical                        | 流复制相关，解码状态                   |
| pg_replslot                       | 流复制相关，复制槽                     |
| pg_serial                         | 事务相关                               |
| pg_snapshots                      | 已导出的快照                           |
| pg_stat                           | 统计信息                               |
| pg_stat_tmp                       | 临时统计信息                           |
| pg_subtrans                       | 子事务                                 |
| pg_tblspc                         | 用于 tablespaces 的链接，指向 base/NNN |
| pg_twophase                       | two phase commmit 相关                 |
| [pg_wal](WAL)                     | WAL 数据                               |
| [pg_xact](XACT)                   | 事务信息                               |
| postgresql.auto.conf              | 配置文件                               |
| postmaster.opts                   | 配置文件                               |
| postmaster.pid                    | 文件锁                                 |


对于 `base/NNN` 内的文件，文件名为表的 filenode。

后缀 `_vm` 为 [VisibilityMap](VisibilityMap) 后缀为 `_fsm` 为 [FreeSapceMap](FreeSapceMap) 后缀为 `_init` 的为 [InitFile](InitFile)。

当一个 PageFile 的大小突破 1G 时就会分裂成多个，命名规则为 `NNN.1` `NNN.2`。

toast 数据与普通表无异。


# PG_VERSION

一个字符串，对于 PostgreSQL 13 来说是 "13\n" (0x31330a)

# PageFile

[doc](https://www.postgresql.org/docs/13/storage-page-layout.html)

每个 PageFile 存储一堆定长的 Page，长度编译时指定一般是 8KiB。

每个 Page 结构如下

| Item           | Description                                                           |
| ----           | -----------                                                           |
| PageHeaderData | 24byte Header                                                         |
| ItemIdData     | 数组，每个 item 对应 15bit offset + 2bit flag + 15 bit length = 32bit |
| Free space     | 未分配的数据                                                          |
| Items          | 实际数据                                                              |
| Special space  | 特殊数据，普通表 0bit                                                 |

所以 PostgreSQL 理论上支持的最大 Block Size 为 (64Ki - 1 ) Byte，实际上为 64KiB。

当 Page Size = 8KiB Tuple = 16Byte 时，Page 层面的空间利用率为 79.9%。

其中 PageHeaderData 如下

| Field               | Type          | Length  | Description                                     |
| -----               | ----          | ------  | -----------                                     |
| pd_lsn              | XLogRecPtr    | 8 bytes | LSN: 修改此 Page 的 xlog 记录的下一个 byte      |
| pd_tli              | uint16        | 2 bytes | 最后一次修改的 TimeLineID                       |
| pd_flags            | uint16        | 2 bytes | Flag bits                                       |
| pd_lower            | LocationIndex | 2 bytes | free space 开始的偏移量                         |
| pd_upper            | LocationIndex | 2 bytes | free space 结束的偏移量                         |
| pd_special          | LocationIndex | 2 bytes | special space 开始的偏移量                      |
| pd_pagesize_version | uint16        | 2 bytes | Page size and layout version number information |
| pd_prune_xid        | TransactionId | 4 bytes | 最旧的以提交 xid (XMAX)                         |

其中 pd_lower 为 ItemIdData 分配的开始，pd_upper 为 Items 分配的开始。

![in page alloc](https://www.postgresql.org/docs/13/pagelayout.svg)

每个 Item （名为 HeapTupleHeaderData）格式如下

| Field       | Type            | Length  | Description                       |
| -----       | ----            | ------  | -----------                       |
| t_xmin      | TransactionId   | 4 bytes | insert XID stamp                  |
| t_xmax      | TransactionId   | 4 bytes | delete XID stamp                  |
| t_cid       | CommandId       | 4 bytes | sql 计数                          |
| t_xvac      | TransactionId   | 4 bytes | VACUUM 的 xid 和 t_cid 是个 union |
| t_ctid      | ItemPointerData | 6 bytes | 行 ID                             |
| t_infomask2 | uint16          | 2 bytes | number of attributes, flag        |
| t_infomask  | uint16          | 2 bytes | flag                              |
| t_hoff      | uint8           | 1 byte  | 用户数据偏移量                    |

ctid = Block Number 4 byte + Offset Number 2 byte

PostgreSQL 为何又存储了一遍 Block ID? + 4 byte

Header 有 23 byte 大。如果只存两个 i64(16byte) 的话，空间利用率为 41%.

Timescale 是认真的么?

# VisibilityMap

[vm](https://www.postgresql.org/docs/13/storage-vm.html)

PostgreSQL 没有说明文件格式，只给了一个[小工具](https://www.postgresql.org/docs/13/pgvisibility.html)说明这个文件工作原理。

一个简单地无 Header 文件，使用一个简单地公式定位

```c
#define HEAPBLK_TO_MAPBLOCK(x) ((x) / HEAPBLOCKS_PER_PAGE)
#define HEAPBLK_TO_MAPBYTE(x) (((x) % HEAPBLOCKS_PER_PAGE) / HEAPBLOCKS_PER_BYTE)
#define HEAPBLK_TO_OFFSET(x) (((x) % HEAPBLOCKS_PER_BYTE) * BITS_PER_HEAPBLOCK)
```

每个 Page 需要至少 2bit 来放 flag

  - 0b01 VISIBILITYMAP_ALL_VISIBLE Page 可见
  - 0b10 VISIBILITYMAP_ALL_FROZEN  Page 冻结

不知道可见和冻结是什么意思。

当这个 page 插入第一个数据时 VISIBILITYMAP_ALL_VISIBLE 被清除

需要看一下 vacuum lazy 如何实现的。

# FreeSapceMap

[fsm](https://www.postgresql.org/docs/13/storage-fsm.html)

[doc](https://github.com/Sasasu/postgres/tree/master/src/backend/storage/freespace)

负责快速找到一个空闲 Page。整体是一个二叉树。

每个 Page 对应 1 byte 空间，存 255 灰度图。

每个 FSM Page 是一个定长的二叉堆，大小与 Block Size 相同（包括 Block Header），默认为 8KiB。

树符合

   $$ value = \max (left, right) $$

```
For example:

    4
 4     2
3 4   0 2    <- This level represents heap pages. 叶子节点保存 Block 的序号
```

整个 FSM 文件使用类似的树形结构。使用定高树，高度为 3。

分配方式为 $$y = n + (n / F + 1) + (n / F^2 + 1) + 1$$

$F$ 为每个 FMS Page 保存的 Block 数量

$n$ 为要寻找的 Block 序号

$y$ 为计算出的 FMS Page 序号

每次插入分配新空间时需要 3 次 `pread` 最坏情况一次全局锁的 `pwrite`，一次 8KiB O(lgN)

每个数据文件大于 1G 时会分裂，3 层的定高树足够用了。

# InitFile

[init](https://www.postgresql.org/docs/13/storage-init.html)

只是一个标记，当存在空表时 PostgreSQL crash 用这个标记来恢复。

# WAL
WAL 与 XLOG 是一个东西。文档不清晰，要翻代码。

WAL 固定大小，16MiB 一个文件，文件依旧无头。

使用的概念上对于一个 TimeLineID 有一个无限长的文件，使用 64bit 的 offset 访问。

文件名格式为 `%08X%08X%08X`
  - TimeLineID 单调递增的启动次数
  - SegmantNo  无限长的 offset 的前 32bit
  - SegmantNo  无限长的 offset 的后 32bit / 文件大小

创建或打开文件在 `XLogFileInit` 中，会循环调用 `pwrite` 写出一个 16MiB 的文件....

后续的写入和读取均使用 `pread` `pwrite`，没有用 `smgr` 的抽象，也都是不定长的随机 IO。

文件读取用 `XLogReaderRoutine` 里的虚函数 `XLogPageRead` 调用 pread 直接读。

写入从共享内存进入 `XLogBackgroundFlush` 里的 `XLogWrite` 里的 pwrite 直接写入。

XLog 有 Block 和 Header，默认情况同样为 8KiB XLOG_BLCKSZ。

引入 Block 大概是为了保存完整性。

XLog 格式大致为 `Header [XLog, XLog]` 每个都是不定长的。解析函数为 `DecodeXLogRecord` 组装函数为 `XLogRecordAssemble`

有空看完 [doc](https://www.interdb.jp/pg/pgsql09.html)

# XACT
