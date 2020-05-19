#### 一条INSERT语句的加锁过程

昨天跑一条INSERT语句想抓一下加锁的调用栈，结果出乎意料的是并没有调用加锁的逻辑。翻了下代码才知道 `InnoDB`里面有个隐式锁的概念。隐式锁和插入意向锁都是`InnoDB`锁子系统的创新点。本文的代码阅读是基于`MySQL 8.0.20`。

##### 1. InnoDB中INSERT语句的执行过程

这里看一条最简单的INSERT语句，即INSERT  INTO ... VALUES类型的。在Server层对一条INSERT语句做完语法解析之后把要插入的值准备好之后会调用handler层的接口来进行实际的数据插入，对`InnoDB`来说，这个接口是`ha_innobase::write_row`。进入 InnoDB之后，一条INSERT语句的调用栈 如下 所示：

``` c++
ha_innobase::write_row()
    row_insert_for_mysql()
    	row_insert_for_mysql_using_ins_graph()
    		row_get_prebuilt_insert_row()
    		row_ins_step()
    			row_ins()
    				row_ins_index_entry_step()
```

上面几个函数的作用如下：

- row_get_prebuilt_insert_row()：用来构建插入操作的运行图，InnoDB通过解释、运行这张图来完成插入操作。
- row_ins_step()：用来给对应的表加IX锁。
- row_ins()：遍历所有索引（即主键索引和二级索引），做索引记录插入。
- row_ins_index_entry_step()：实际用来进行索引插入的函数。

##### 2. 关键函数分析

###### 2.1 row_ins_clust_index_entry_low()  插入聚簇索引的总流程

`row_ins_clust_index_entry_low()`函数用来插入聚簇索引记录，它首先会检查要插入的记录是否已经存在，接着再进行记录的插入。函数的注释代码如下：

``` c++
dberr_t row_ins_clust_index_entry_low()
{
    //1. 开启一个mini transaction.
    mtr.start();
    //2. 将游标定位到要插入的位置；
    btr_pcur_open(index, entry, PAGE_CUR_LE, mode, &pcur, &mtr);
    //3. 检查是否存在相同的主键记录,如果存在，则尝试对该记录加S锁，并返回错误码。
    //如果加锁失败，则表示插入这条记录的事务还没提交，这时函数会返回DB_LOCK_WAIT.
    err = row_ins_duplicate_error_in_clust(flags, cursor, entry, thr, &mtr);
    if (err != DB_SUCCESS) {
        //4. 如果有重复记录，或者存在LOCK_WAIT，则返回错误。
        goto func_exit;
    }
    // 5. 进行乐观/悲观插入，注意这一步必要的话创建锁，接着创建UNDO日志。
    btr_cur_optimistic_insert()/btr_cur_pessimistic_insert();
    // 6. 提交mini transaction.
    mtr.commit();
funct_exit:
    return(err);
}
```

###### 2.2 row_ins_duplicate_error_in_clust() 检查是否有重复记录，如果有的话对该记录加锁

2.1节对函数`row_ins_clust_index_entry_low()`的注释中，第3步是检查是否有相同主键的记录。如果没有相同的记录，则返回DB_SUCCESS，否则返回错误码，错误码可能是`DB_LOCK_WAIT`或者是`DB_DUPLICATE_KEY`。下面看一下`row_ins_duplicate_error_in_clust()`的流程：

``` c++
static dberr_t
row_ins_duplicate_error_in_clust(
        ulint flags,           /*!< in: undo logging and locking flags */
        btr_cur_t *cursor,     /*!< in: B-tree cursor */
        const dtuple_t *entry, /*!< in: entry to insert */
        que_thr_t *thr,        /*!< in: query thread */
        mtr_t *mtr)            /*!< in: mtr */
{
  n_unique = dict_index_get_n_unique(cursor->index);
  // 1. 如果存在相同记录
  if (cursor->low_match >= n_unique || cursor->up_match >= n_unique) {
      // 2. 对该记录尝试进行加锁， err可能返回DB_SUCESS或DB_SUCCESS_LOCKED_REC
      //或者是 DB_LOCK_WAIT
      err = row_ins_set_shared_rec_lock(lock_type, btr_cur_get_block(cursor),
                                          rec, cursor->index, offsets, thr)
      // 3. 根据是否加锁成功，来决定：如果加锁成功则，判断记录是否被删除，如果失败，则直接返回
      switch (err) {
        case DB_SUCCESS_LOCKED_REC:
        case DB_SUCCESS:
          break;
        default:
          goto func_exit;
      }
      // 4. 加锁成功，查看这个相同的记录是否是有效记录（是否打了deleted mark）。
      if (row_ins_dupl_error_with_rec(rec, entry, cursor->index, offsets)) {
        // 5. 表示相同的记录是有效记录，则这里返回重复主键错误（DB_DUPLICATE_KEY）。
      duplicate:
        trx->error_info = cursor->index;
        err = DB_DUPLICATE_KEY;
        goto func_exit;
      }
      // 6.走到这里说明没有相同主键的记录，或者存在相同的记录但是记录已经被打上deleted mark了
  }
  err = DB_SUCCESS;
func_exit:
  return (err);
}
```

考虑一下这个场景：

``` mysql
mysql> select * from t2;
+----+------+
| pk | data |
+----+------+
|  1 |    2 |
|  2 |    3 |
| 10 |   11 |
+----+------+
3 rows in set (0.01 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t2 values(10,11);
ERROR 1062 (23000): Duplicate entry '10' for key 'PRIMARY'

```

在14行，虽然INSERT返回重复主键错误，但是事务对pk=10这条记录的锁仍然是持有的，因为两阶段锁协议要保证在事务提交之前，不能释放所持有的锁。所以，当用另外一个客户端查询pk=10这条记录时会出现锁冲突而卡住：

``` mysql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update t1 set pk=95 where pk=110;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

###### 2.3 btr_cur_optimistic_insert() 进行聚簇索引记录的插入，写UNDO记录并加锁

在2.2节对记录进行唯一性检测通过之后开始插入记录。实际的插入之前需要检测是否需要加锁，还要写入UNDO日志。这些都是在`btr_cur_optimistic_insert()`函数中进行的（如果插入导致了B+树分裂，则会调用`btr_cur_pessimistic_insert()`进行悲观插入），函数的注释版代码如下：

``` c++
dberr_t btr_cur_optimistic_insert() {
     const rec_t *page_cursor_rec = page_cur_get_rec(page_cursor);

    /* 1. 检查是否有GAP锁禁止插入，如果检查通过，则说明可以插入，这时写入UNDO日志。
    注意：如果游标所在的记录没有锁，则下面这个函数不会创建任何锁，直接返回（使用隐式
    锁机制）。如果有锁冲突，则会创建一个插入意向锁，加入到等待队列，并返回DB_LOCK_WAIT。*/
    err = btr_cur_ins_lock_and_undo(flags, cursor, entry, thr, mtr, &inherit);

    if (err != DB_SUCCESS) {
        goto fail_err;
    }
	/* 2. 将记录插入到游标所在的位置。 */
    *rec = page_cur_tuple_insert(page_cursor, entry, index, offsets, heap,
                                   n_ext, mtr);
}
```

###### 2.4 lock_rec_insert_check_and_lock 锁检测的与创建

2.3节中代码注释的第1条`btr_cur_ins_lock_and_undo()`会直接调用`lock_rec_insert_check_and_lock ()`来检查锁冲突，并在出现锁冲突时，创建插入意向锁。该函数的注释版代码如下所示：

``` c++
dberr_t lock_rec_insert_check_and_lock(
    ulint flags,         /*!< in: if BTR_NO_LOCKING_FLAG bit is
                         set, does nothing */
    const rec_t *rec,    /*!< in: record after which to insert */
    buf_block_t *block,  /*!< in/out: buffer block of rec */
    dict_index_t *index, /*!< in: index */
    que_thr_t *thr,      /*!< in: query thread */
    mtr_t *mtr,          /*!< in/out: mini-transaction */
    ibool *inherit)      /*!< out: set to true if the new
                         inserted record maybe should inherit
                         LOCK_GAP type locks from the successor
                         record */
{
  lock_mutex_enter();
  
  // 1. 首先查看游标所在的位置是否有锁。
  lock = lock_rec_get_first(lock_sys->rec_hash, block, heap_no);

  // 2. 如果没有锁，则直接返回成功。
  if (lock == NULL) {
    /* We optimize CPU time usage in the simplest case */

    lock_mutex_exit();
    return (DB_SUCCESS);
  }

  *inherit = true;

  // 3. 设置要加的锁模式为X类型的插入意向锁
  const ulint type_mode = LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION;

  // 4. 检查要加的插入意向锁与当前的锁是否冲突
  const lock_t *wait_for =
      lock_rec_other_has_conflicting(type_mode, block, heap_no, trx);

  if (wait_for != NULL) {
    // 5. 如果有冲突，则创建一个插入意向锁，加入到当前锁的等待队列中。
    RecLock rec_lock(thr, index, block, heap_no, type_mode);
    err = rec_lock.add_to_waitq(wait_for);
  } else {
    // 6. 如果没有冲突，则直接返回。
    err = DB_SUCCESS;
  }

  lock_mutex_exit();
  return (err);
}
```

##### 总结

通过上面函数的阅读，可以看到一条INSERT语句可能会在两个地方加锁：第一在检查是否存在和要插入记录相同的记录时，如果存在相同记录，则会对该记录上S锁；第二，在进行记录插入时，会检查游标所在的记录是否有GAP锁，如果有的话则会创建一个插入意向锁，放入到当前锁的等待队列，如果没有，则不会创建任何锁，直接返回。

