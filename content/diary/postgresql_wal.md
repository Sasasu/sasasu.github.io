---
title: "PostgreSQL 的 WAL"
date: 2021-11-07T21:51:36+08:00
draft: true
---

PostgreSQL 的块 IO 大小是 8KiB (BLOCKSIZE)，对于 8KiB 的文件使用 `sync_file_range()` 并不是原子的。

为了保证原子性 PostgreSQL 使用借助 WAL，每次事务提交前将 WAL 刷盘，事务结束。崩溃时从上一个存档点将所有已提交 WAL 重做，恢复到崩溃前的状态。

PostgreSQL 的 WAL 目前只有 REDO LOG，也就是说 PostgreSQL 不会修改旧数据，会将版本链在数据文件中构建。频繁的做 `update FOO set bar=baz` 会导致数据文件膨胀，需要使用 `VACCUM FULL` 来重写整个数据文件。

EnterpriseDB 有一个支持 UNDO LOG 的分支，不需要频繁的 `VACCUM FULL`，这个数据格式叫 [zheap](https://github.com/EnterpriseDB/zheap) 目前处于弃坑状态中。

---

REDO LOG 的 `fsync()` 略微复杂。

首先有一个配置字段 `wal_buffers`，支持各种奇葩单位，换算后的值为 8KiB (XLOG_BLCKSZ) ~ 16MiB (Wal Segment Size).
当有新的 WAL 到达时需先用 `XLogInsert()` 和 `XLogInsertRecord()` 将数据拷贝到 `wal_buffers` 中。
`XLogInsertRecord()` 的代码面条风格严重，`write offset` `buffer` `segment` 这三层没有分离，完全挤在一起。
其中分配新的 buffer 的逻辑在 `XLogInsertRecord() -> CopyXLogRecordToWAL() -> GetXLogBuffer() -> AdvanceXLInsertBuffer()`
其中最顶层的 `XLogInsertRecord()` 进入时需要全局锁。`AdvanceXLInsertBuffer()` 调用了 `WaitXLogInsertionsToFinish()` 它遍历所有的写锁，唤醒 WAL 刷盘进程，等刷盘完成后返回。

总结来说在正常写入流程中会触发 buffer 分配和旧 buffer 刷盘(带着 fsync)。吞吐量大的时候一定会在这里卡一下。

当进行 `commit` 时还有另一个配置字段 `synchronous_commit`。入口为 `XactLogCommitRecord()` 会直接调用 `XLogFlush()` 它会先等锁 `WaitXLogInsertionsToFinish()` 然后 `XLogWrite()` 来对文件进行 `fsync`。
调用路径上没有明显区分是否需要 `fsync` 靠上层设置一些诡异参数，下层根据这些参数来判断是否需要 `fsync`。同时有很多实际逻辑比较诡异。比如可以显著提升吞吐量的 `CommitDelay`。

GPDB 不支持 `synchronous_commit=off`，因为 2PC 需要读 WAL。需要落盘。CLOG 在 commit 时不持久化，因为可以从 WAL 中恢复。

WAL 的写入都由 checkpointer 后台进程发起，会不会有 wait free 的效果值得怀疑。checkpointer 每 `checkpoint_timeout` 创建一个存档点，同时将 WAL 刷盘。

---

值得一提的是，按照常规逻辑，每次 checkpointer 之后的第一次修改，应该会产生一个 `full page write` 将整个 heap 表的数据块在 WAL 中保存一份。
因为在恢复开始时 PostgreSQL 无法确认这个数据块的初始状态，直接按偏移量重做写入操作会出现非预期的最终状态。

PostgreSQL 使用 2 个 GUC 来控制这个行为。`full_page_writes` 和 `wal_log_hints`

`full_page_writes` 启用时，PostgreSQL 会在存档点之后的第一次修改数据块时产生 full_page_write。

`wal_log_hints` 启用时，PostgreSQL 在存档点之后第一次修改数据块且仅修改事务提示位时也会产生 full_page_write。事务提示位即为 wal_log_hint，如果为 1 则此块中的所有数据均为以提交。

`full_page_writes` 默认启用，`wal_log_hints` 默认不启用。

## 草稿

### 文件部分
PostgreSQL WAL 预分配是用循环写 0
PostgreSQL 的 WAL 只支持全物理记录和全逻辑记录，不支持页内逻辑。
WAL 没有结束标记，每次读到末尾时靠报错重试。
WAL 没有良好的二进制格式，无法并行解码。

### 复制部分
PostgreSQL 的主从同步同样需要 WAL。使用硬盘作为 MQ 来做 IPC。
logic decode 和 physical decode. 没有单独的 binlog 文件。不能做「备份 logic WAL」 这种操作
decode 复用 crash-recovery 部分的逻辑，使得第三方工具制作难度高
