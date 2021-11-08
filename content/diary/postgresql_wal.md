---
title: "PostgreSQL 的 WAL"
date: 2021-11-07T21:51:36+08:00
draft: false
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

`full_page_writes` 启用时，PostgreSQL 会在存档点之后的第一次修改数据块时产生 `full_page_write`。

`wal_log_hints` 启用时，PostgreSQL 在存档点之后第一次修改数据块且仅修改事务提示位时也会产生 `full_page_write`。事务提示位即为 `wal_log_hint`，如果为 1 则此块中的所有数据均为以提交。

`full_page_writes` 默认启用，`wal_log_hints` 默认不启用。

---

PostgreSQL 有 WAL 预分配，当 `wal_init_zero` 为 on 时会使用阻塞的 `pwrite()` 来循环写 0 填充文件，在 PostgreSQL 13 上会使用 `writev` 但每个 `syscall` 只写入一个 `Block`，与 `pwrite` 无异。

当 `wal_init_zero` 为 off 时会使用 `pwrite()` 到超过文件末尾来产生一个 8KiB 的空洞填充文件。没有使用 `fallocate()` 这样的 `syscall`。

```c
if (wal_init_zero) {
    /*
     * Zero-fill the file.  With this setting, we do this the hard way to
     * ensure that all the file space has really been allocated.  On
     * platforms that allow "holes" in files, just seeking to the end
     * doesn't allocate intermediate space.  This way, we know that we
     * have all the space and (after the fsync below) that all the
     * indirect blocks are down on disk.  Therefore, fdatasync(2) or
     * O_DSYNC will be sufficient to sync future writes to the log file.
     */
    for (nbytes = 0; nbytes < wal_segment_size; nbytes += XLOG_BLCKSZ) {
        errno = 0;
        if (write(fd, zbuffer.data, XLOG_BLCKSZ) != XLOG_BLCKSZ) {
            /* if write didn't set errno, assume no disk space */
            save_errno = errno ? errno : ENOSPC;
            break;
        }
    }
} else {
    /*
     * Otherwise, seeking to the end and writing a solitary byte is
     * enough.
     */
    errno = 0;
    if (pg_pwrite(fd, zbuffer.data, 1, wal_segment_size - 1) != 1) {
        /* if write didn't set errno, assume no disk space */
        save_errno = errno ? errno : ENOSPC;
    }
}
```

`wal_init_zero` 的默认值是 on

---

PostgreSQL 只有一种 WAL，不像 MySQL 一样有与数据文件无关的 binlog。

这种日志是且仅是物理日志（physical WAL）。在日志中直接记录了块内偏移量。没有采用块内逻辑日志这种易于升级的方案。

```c
typedef struct xl_heap_update {
    TransactionId old_xmax;       /* xmax of the old tuple */
    OffsetNumber old_offnum;      /* old tuple's offset */
    uint8       old_infobits_set; /* infomask bits to set on old tuple */
    uint8       flags;
    TransactionId new_xmax;       /* xmax of the new tuple */
    OffsetNumber new_offnum;      /* new tuple's offset */
} xl_heap_update;
```

这是 `UPDATE` 的 redo log，PostgreSQL 在恢复时根据 `old_offnum` 找出旧的 `tuple` 修改 `xmax` 然后将 WAL 记录中的新 `tuple` 放置到 `new_offnum` 中。

日志中没有任何元数据，仅有偏移量和丢失类型信息的数据。且数据不定长，解码需要单线程，不能随机读某个 WAL。

WAL 没有结束标志，又一定会使用 0 填充。有些工具总会报出 `incorrect length 0` 就是因为此。

同时 WAL sender 与 WAL writer 使用硬盘作为消息队列互相通讯，WAL writer 先写到硬盘，WAL sender 在从硬盘中读取。这要求 WAL sender 一定要从硬盘上读取文件，然后会遇到错误。

解决这个问题的方式是不断的重试。WAL 损坏时 PostgreSQL 死循环无法启动便是因为这个设计。

---

PostgreSQL 虽然只有 「physical WAL」 但是依然可以做逻辑复制。原因是在「logical replication」中，WAL Sender 会多一步特殊的 `decode`。见 `src/backend/replication/logical/decode.c`

在这种 `decode` 过程中需要根据偏移量查找出元信息，然后将元信息与 WAL 重新组装成为可以逻辑复制的东西，再发送到其他 PostgreSQL 服务器里。查找元信息的过程是一次内存中的 hash 表查找。

所以 PostgreSQL 并没有备份 `binlog` 这种操作。某个机器产生的 WAL 文件只能和与其进行 `physical replication` 的机器共用。


同时 PostgreSQL 的解码过程十分依赖内部的工具函数 `XLogReader`。`pg_waldump` `wal sender` `recovery` 等都使用了 `XLogReader`。其中有大量的特定日志解析代码。
这使得离开 PostgreSQL 的主代码库来写利用 WAL 的工具比较困难，PostgreSQL 也没有详细说明每种 WAL 的二进制格式，严重依赖 C struct 的 ABI。

所以很少有人对 PostgreSQL 的 WAL 做什么事情，一般是在旁边挂上一个接受 logical replication 的 agent，间接获取 binlog 来做复制或者备份。
