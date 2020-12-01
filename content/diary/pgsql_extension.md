---
title: "pgsql_extension"
date: 2020-11-27T23:23:34+08:00
draft: true
---

# PostgreSQL 的扩展

pgsql 号称高扩展性，实际上单纯是是在关键路径上有很多函数指针，pgsql 使用 dlopen 之后允许用户程序修改这些指针而已。

可以在代码里搜 `extern PGDLLIMPORT` 和 `hook` 找到所有的扩展点。

```
include/utils/elog.h:412:extern PGDLLIMPORT emit_log_hook_type emit_log_hook;
include/utils/lsyscache.h:63:extern PGDLLIMPORT get_attavgwidth_hook_type get_attavgwidth_hook;
include/utils/selfuncs.h:124:extern PGDLLIMPORT get_relation_stats_hook_type get_relation_stats_hook;
include/utils/selfuncs.h:129:extern PGDLLIMPORT get_index_stats_hook_type get_index_stats_hook;
include/tcop/utility.h:76:extern PGDLLIMPORT ProcessUtility_hook_type ProcessUtility_hook;
include/fmgr.h:771:extern PGDLLIMPORT needs_fmgr_hook_type needs_fmgr_hook;
include/fmgr.h:772:extern PGDLLIMPORT fmgr_hook_type fmgr_hook;
include/catalog/objectaccess.h:132:extern PGDLLIMPORT object_access_hook_type object_access_hook;
include/commands/explain.h:72:extern PGDLLIMPORT ExplainOneQuery_hook_type ExplainOneQuery_hook;
include/commands/explain.h:76:extern PGDLLIMPORT explain_get_index_name_hook_type explain_get_index_name_hook;
include/commands/user.h:25:extern PGDLLIMPORT check_password_hook_type check_password_hook;
include/libpq/auth.h:27:extern PGDLLIMPORT ClientAuthentication_hook_type ClientAuthentication_hook;
include/libpq/libpq-be.h:293:extern PGDLLIMPORT openssl_tls_init_hook_typ openssl_tls_init_hook;
include/parser/analyze.h:22:extern PGDLLIMPORT post_parse_analyze_hook_type post_parse_analyze_hook;
include/rewrite/rowsecurity.h:40:extern PGDLLIMPORT row_security_policy_hook_type row_security_policy_hook_permissive;
include/rewrite/rowsecurity.h:42:extern PGDLLIMPORT row_security_policy_hook_type row_security_policy_hook_restrictive;
include/optimizer/plancat.h:25:extern PGDLLIMPORT get_relation_info_hook_type get_relation_info_hook;
include/optimizer/paths.h:33:extern PGDLLIMPORT set_rel_pathlist_hook_type set_rel_pathlist_hook;
include/optimizer/paths.h:42:extern PGDLLIMPORT set_join_pathlist_hook_type set_join_pathlist_hook;
include/optimizer/paths.h:48:extern PGDLLIMPORT join_search_hook_type join_search_hook;
include/optimizer/planner.h:30:extern PGDLLIMPORT planner_hook_type planner_hook;
include/optimizer/planner.h:38:extern PGDLLIMPORT create_upper_paths_hook_type create_upper_paths_hook;
include/executor/executor.h:66:extern PGDLLIMPORT ExecutorStart_hook_type ExecutorStart_hook;
include/executor/executor.h:73:extern PGDLLIMPORT ExecutorRun_hook_type ExecutorRun_hook;
include/executor/executor.h:77:extern PGDLLIMPORT ExecutorFinish_hook_type ExecutorFinish_hook;
include/executor/executor.h:81:extern PGDLLIMPORT ExecutorEnd_hook_type ExecutorEnd_hook;
include/executor/executor.h:85:extern PGDLLIMPORT ExecutorCheckPerms_hook_type ExecutorCheckPerms_hook;
include/storage/ipc.h:78:extern PGDLLIMPORT shmem_startup_hook_type shmem_startup_hook;
```

# utils
[emit_log_hook](https://github.com/Sasasu/postgres/blob/0a4db67b5ed05c4013ea968930af36853f088404/src/include/utils/elog.h#L412) 处于 `erereport` 和 `elog` 的处理路径上。
```c
void emit_log_hook(ErrorData *edata);
```
应该可以用来修改错误级别，审计错误之的，没见过有人加这个 hook。

[get_attavgwidth_hook](https://github.com/Sasasu/postgres/blob/0a4db67b5ed05c4013ea968930af36853f088404/src/include/utils/lsyscache.h#L62) 优化器相关，可以直接覆盖 `get_attavgwidth`。
```c
int32 get_attavgwidth_hook(Oid relid, AttrNumber attnum);
```
获取表中某一列的平均宽度。对于变长类型有用。

[get_relation_stats_hook](https://github.com/Sasasu/postgres/blob/0a4db67b5ed05c4013ea968930af36853f088404/src/include/utils/lsyscache.h#L62) 优化器相关。
```c
bool get_relation_info_hook(PlannerInfo *root, RangeTblEntry *rte, AttrNumber attnum, VariableStatData *vardata);
```

# IPC
[shmem_startup_hook](https://github.com/Sasasu/postgres/blob/0a4db67b5ed05c4013ea968930af36853f088404/src/include/storage/ipc.h#L78) PostgreSQL 的 IPC 一部分。用于在 `postmaster` 上开辟共享内存区域，与其他进程通讯。
```c
void shmem_startup_hook_type(void);
```
在 `postmaster` 和 `bgworker` 上都会运行这个函数。

在 attach 共享内存之前需要在 `void _PG_init()` 里使用 `void RequestAddinShmemSpace(Size)` 分配共享内存。

释放内存是自动的，PostgreSQL 也没有暴露通过 IPC 发送一个指针子进程再去 `memopen` 的操作。

对于内存已外资源有 `on_shmem_exit` hook 可以析构内存以外的资源。

