---
title: mysql.gtid_executed table persistence
date: 2024-06-29
categories: CS
tags:
- mysql
- database
---

*Here is a doc capturing how gtid_executed list in memory is updated upon transactions, when/how that list is flushed into mysql.gtid_executed table, relationship between persistence and undo purge thread, its read/write scenarios, how innodb/rocksdb treats that differently.*

### trx & gtid_executed list in memory

This part is only executed in InnoDB. When a trx is in the prepared stage in InnoDB, it allocates an update undo segment if it is an insert only transaction. This is because the insert undo segment can be purged immediately(because no snapshot is reading it), but we need to persist the GTID information before purging. For the update undo segment, it will only be purged after being persisted to the `gtid_executed` table. After allocating the segment, it will write the gtid version and value to the undo header.

```c++
// assign update undo segment
trx_commit_for_mysql(trx_t *trx):
    case TRX_STATE_PREPARED:
        trx->op_info = "committing";

        /* For GTID persistence we need update undo segment. */
        db_err = trx_undo_gtid_add_update_undo(trx, false, false);
        
trx_undo_gtid_add_update_undo:
  if (undo_ptr->is_insert_only() || gtid_explicit) {
    mutex_enter(&trx->undo_mutex);
    db_err = trx_undo_assign_undo(trx, undo_ptr, TRX_UNDO_UPDATE);
    mutex_exit(&trx->undo_mutex);
  }

// write gtid to undo
trx_undo_gtid_write:
  if (gtid_desc.m_is_set) {
    /* Persist gtid version */
    mlog_write_ulint(undo_header + TRX_UNDO_LOG_GTID_VERSION,
                     gtid_desc.m_version, MLOG_1BYTE, mtr); 
    /* Persist fixed length gtid */
    ut_ad(TRX_UNDO_LOG_GTID_LEN == GTID_INFO_SIZE);
    mlog_write_string(undo_header + gtid_offset, &gtid_desc.m_info[0],
                      TRX_UNDO_LOG_GTID_LEN, mtr); 
    undo->flag |= gtid_flag;
  }
```

Later when committing the trx in memory, it will add the gtid value to a `Gitd_info_list` that is active when running transactions. If the number of GTID accumulated in memory exceeds the threshold, flush that. It does that by setting `m_event`, for which `Clone_persist_gtid::periodic_write()` waits for.

```
|-trx_commit_in_memory
  |-trx_release_impl_and_expl_locks
    |-Clone_persist_gtid::add
      /* Get active GTID list */
  		auto &current_gtids = get_active_list();

  		/* Add input GTID to the set */
  		current_gtids.push_back(gtid_desc.m_info);
  		if (current_value == s_gtid_threshold) {
        os_event_set(m_event);
      }
      |-Clone_persist_gtid::periodic_write()
  	  |-Clone_persist_gtid::wait_flush
  	    |-Clone_persist_gtid::wait_thread
```

### mysql.gtid_executed persistence

**Some important members of Clone_persist_gtid:**

`m_gtid_trx_no` Oldest transaction number for which GTID is not persisted

`m_num_gtid_mem` Number of GTID accumulated in memory

` Gitd_info_list m_gtids[2]` Two lists of GTID. One of them is active where running transactions add their GTIDs. Other list is used to persist them to table from time to time. Getting which list is depending on on `m_active_number`, which is incremented upon flush.

```c++
Gitd_info_list &get_active_list() {
    ut_ad(trx_sys_serialisation_mutex_own());
    // when m_active_number is incremented, the list is switched due to &static_cast<uint64_t>(1) logic in get_list
    return (get_list(m_active_number));
  }
  
Gitd_info_list &get_list(uint64_t list_number) {
    int list_index = (list_number & static_cast<uint64_t>(1));
    return (m_gtids[list_index]);
  }
```

`  std::atomic<uint64_t> m_active_number` Number of the current GTID list. Increased when list is switched

`std::atomic<uint64_t> m_flush_number` Number up to which GTIDs are flushed. Increased when list is flushed

`  const static int s_gtid_threshold = 1024` Number of transaction/GTID threshold for writing to disk table

`const static int s_max_gtid_threshold = 1024 * 1024` Maximum Number of transaction/GTID to hold. Transaction commits 

must wait beyond this point. Not expected to happen as GTIDs are compressed and written together.

`os_event_t m_event` Event for GTID background thread

`s_time_threshold` Time(millisecond) threshold to trigger persisting GTID

**Flush & purge:**

```
|-Clone_persist_gtid::periodic_write
  |-Clone_persist_gtid::flush_gtids
    |-Clone_persist_gtid::write_to_table
      |-Gtid_table_persistor::save
    |-Clone_persist_gtid::update_gtid_trx_no
      |-void trx_sys_persist_gtid_num
    |-Gtid_table_persistor::compress
```

`Clone_persist_gtid::flush_gtids` is the core function for gtid_executed table persistence. Although RocksDB would start the `ib_clone_gtid` thread and this function is also called, it will effectively do nothing because in the previous step, the gtid list in memory is not appended, and `m_num_gtid_mem` will always be 0. It is called every 100ms, or gtid threshold > 1024. 

For InnoDB, If there is any accumulated GTID in memory, it will start the flush process, get the flush list, save into the `gtid_executed` table. Then it updates `m_gtid_trx_no` which represents the oldest transaction number for which GTID is not persisted to be the next trx_no of latest trx_no that is just flushed, and write to the system header page, then wake up the undo purge thread, do compression if necessary.

In the purge thread, it will not purge the undo log if that's not persisted into the gtid_executed table. See the function that determines the oldest view:

```c++
void MVCC::clone_oldest_view(ReadView *view) {
  trx_sys_mutex_enter();
  ReadView *oldest_view = get_oldest_view();
  if (oldest_view == nullptr) {
    view->prepare(0);
    trx_sys_mutex_exit();
  } else {
    view->copy_prepare(*oldest_view);
    trx_sys_mutex_exit();
    view->copy_complete();
  }
  /* Update view to block purging transaction till GTID is persisted. */
  auto &gtid_persistor = clone_sys->get_gtid_persistor();
  auto gtid_oldest_trxno = gtid_persistor.get_oldest_trx_no();
  view->reduce_low_limit(gtid_oldest_trxno);
}

/** Check and reduce low limit number for read view. Used to
block purge till GTID is persisted on disk table.
@param[in]    trx_no  transaction number to check with */
void reduce_low_limit(trx_id_t trx_no) {
  if (trx_no < m_low_limit_no) {
    /* Save low limit number set for Read View for MVCC. */
    ut_d(m_view_low_limit_no = m_low_limit_no);
    m_low_limit_no = trx_no;
  }
}
```

### Write/read Scenarios

- **Write Scenarios**:
  - [server start](https://github.com/facebook/mysql-5.6/blob/fb-mysql-8.0.32/sql/mysqld.cc#L9099)
  - [trx commit in slave with binlog=OFF](https://github.com/facebook/mysql-5.6/blob/fb-mysql-8.0.32/sql/binlog.cc#L2101)
  - [committing a statement or transaction, including XA, and also at XA prepare handling](https://github.com/facebook/mysql-5.6/blob/fb-mysql-8.0.32/sql/handler.cc#L1501)
  - [server shutdown](https://github.com/facebook/mysql-5.6/blob/fb-mysql-8.0.32/sql/mysqld.cc#L9474)
  - [binlog rotation](https://github.com/facebook/mysql-5.6/blob/fb-mysql-8.0.32/sql/binlog.cc#L8859)
  - [flush_gtid](https://github.com/facebook/mysql-5.6/blob/fb-mysql-8.0.32/storage/innobase/clone/clone0repl.cc#L478). Called upon server start, reaching time threshold(100ms) or gtid threshold(1024)). This has an effect on InnoDB only

  **Read Scenarios:**
  - [server start](https://github.com/facebook/mysql-5.6/blob/fb-mysql-8.0.32/sql/mysqld.cc#L9023)
  - [clone recovery](https://github.com/facebook/mysql-5.6/blob/fb-mysql-8.0.32/storage/innobase/clone/clone0repl.cc#L527)

### References

- https://blog.csdn.net/weixin_34238642/article/details/90065665
- https://cloud.tencent.com/developer/article/2083743
- http://mysql.taobao.org/monthly/2023/11/02/
- https://blog.csdn.net/n88Lpo/article/details/127002441
- https://zhuanlan.zhihu.com/p/141403577
- https://cloud.tencent.com/developer/article/1396314