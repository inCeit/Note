### MySQL 打开一张表的流程

在做特性测试的时候，在一些表空间丢失或损坏的异常场景下，server层会报表不存在或表空间missing。当时只关注了`InnoDB`层的`ha_innobase::open`函数，对整个打开表的过程了解不是很清楚，现在再回头确认一下这些处理流程。搞清楚下面的问题：

- 哪种情况返回`ER_NO_SUCH_TABLE`或`ER_TABLESPACE_MISSING`，差异之处在哪里。
- server层表管理的逻辑，包括handler的生成，TABLE对象的生成，什么是TABLESHARE等。

#### 1. `InnoDB`打开一张表

`InnoDB`的handler对象是`ha_innobase`，它继承了server层的`handler`对象。在打开表时server层会调用`ha_innobase::open`这个接口函数来打开`InnoDB`中的一张表。原来以为server层的表对象会跟innodb的表对象在`ha_innobase::open`的时候会建立某种联系，看了一下两者其实没啥联系，`ha_innobase::open`只是会在`InnoDB`层准备好server层要访问的表。

`ha_innobase::open`函数精简后的代码如下：

``` c++
int
ha_innobase::open(
/*==============*/
	const char*		name,		/*!< in: table name */
	int			mode,		/*!< in: not used */
	uint			test_if_locked)	/*!< in: not used */
{
	dict_table_t*		ib_table;

	thd = ha_thd();

	m_user_thd = NULL;

	if (!(m_share = get_share(name))) {

		DBUG_RETURN(1);
	}

	/* 对于intrinsic的临时表，innodb不会放入数据字典，而是会放入到线程局部变量中。
    对于其他表innodb需要从数据字典中打开。*/
	ib_table = thd_to_innodb_session(thd)->lookup_table_handler(norm_name);

	if (ib_table == NULL) {
		/* 1. 正常表从数据字典打开。 */
		ib_table = open_dict_table(name, norm_name, is_part,
					   ignore_err);
	} else {
        /* 2. intrinsic的表，直接递增引用计数。 */
		ib_table->acquire();
	}

	if (ib_table != NULL
	    && dict_table_is_encrypted(ib_table)
	    && ib_table->ibd_file_missing
	    && !dict_table_is_discarded(ib_table)) {

		dict_table_close(ib_table, FALSE, FALSE);
		/* 对于加密表如果表空间不存在，则会首先返回ER_CANNOT_FIND_KEY_IN_KEYRING，
        而不是IBD missing，这里需要注意。my_error是server层的函数，用来发送错误信息
        给客户端。*/
        my_error(ER_CANNOT_FIND_KEY_IN_KEYRING, MYF(0));

		DBUG_RETURN(HA_ERR_TABLE_CORRUPT);
	}

    /* 如果innodb层没有成功生成dict_table_t表对象，则会返回ER_NO_SUCH_TABLE。 */
	if (NULL == ib_table) {
		set_my_errno(ENOENT);

		DBUG_RETURN(HA_ERR_NO_SUCH_TABLE);
	}

	if (dict_table_is_discarded(ib_table)) {
        /* 如果是IBD文件被用户使用ALTER TABLE DISCARD TABLESPACE语句删除了，
        则会给出警告：ER_TABLESPACE_DISCARDED，注意这里不是错误，因为是用户主动
        删的IBD文件*/
		ib_senderrf(thd,
			IB_LOG_LEVEL_WARN, ER_TABLESPACE_DISCARDED,
			table->s->table_name.str);
	} else if (ib_table->ibd_file_missing) {
		/* 如果表的IBD文件意外丢失了，则会返回错误ER_TABLESPACE_MISSING。 */
		ib_senderrf(
			thd, IB_LOG_LEVEL_WARN,
			ER_TABLESPACE_MISSING, norm_name);
		no_tablespace = true;
	} else {
		no_tablespace = false;
	}

	if (!thd_tablespace_op(thd) && no_tablespace) {
		set_my_errno(ENOENT);

		dict_table_close(ib_table, FALSE, FALSE);

		DBUG_RETURN(HA_ERR_TABLESPACE_MISSING);
	}

    /* 下面的代码和我们要关注的点联系不大，就不看了。 */
	m_prebuilt = row_create_prebuilt(ib_table, table->s->reclength);

	if (!innobase_build_index_translation(table, ib_table, m_share)) {
		  sql_print_error("Build InnoDB index translation table for"
				  " Table %s failed", name);
	}

	dberr_t	err = fil_set_compression(m_prebuilt->table,
					  table->s->compress.str);

	DBUG_RETURN(0);
}
```

#### 2. Server打开一张表

在server层，要打开一张表需要创建一个`handler`。比如，如果要打开一张`InnoDB`的表，需要实例化一个`ha_innobase`对象，`ha_innobase`类型是`hanlder`的子类。所有对这个表的访问都是通过`ha_innobase`对象的公共方法来进行的。通过调用`handler *get_new_handler(TABLE_SHARE *share, MEM_ROOT *alloc, handlerton *db_type)`server层的线程获取一个handler实例。对于`InnoDB`来说，上面的函数想当于调用`innobase_create_handler`函数来初始化一个`ha_innobase`类型的实例。拿到实例化后的`ha_innobase`对象后，就可以用这个对象来访问表了。

Server层打开一张表最终会走到上面的`ha_innobase::open`函数。下面是一个`SELECT`语句执行中打开表的调用顺序：

``` c++
mysql_execute_command()
    execute_sqlcom_select()
    	open_tables_for_query()
    		open_tables()
    			open_and_process_table()
    				open_table()
    					open_table_from_share()
    						handler::ha_open()
    						ha_innobase::open()
```

上面的函数中最重要的函数是`open_table_from_share`，这个函数会创建`handler::table`成员，并从`TABLE_SHARE`对象中初始化`table`成员。`handler::table`在访问表中的记录时会使用到，例如存放读取的记录，暂存要写入的记录等等。

##### 2.1 从TABLE_SHARE
