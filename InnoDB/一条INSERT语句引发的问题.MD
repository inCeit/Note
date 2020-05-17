#### 一条INSERT语句引发的思考

昨天跑一条INSERT语句想抓一下加锁的调用栈，结果出乎意料的是并没有调用加锁的逻辑。翻了下代码才知道 `InnoDB`里面有个隐式锁的概念。本文的代码阅读是基于`MySQL 8.0.20`。

##### InnoDB中INSERT语句的执行过程

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

