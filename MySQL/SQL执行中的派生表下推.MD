### SQL执行中的派生表下推

一直想搞清楚对于带子查询的`JOIN`是如何进行优化的，于是跟踪了一下下面这条SELECT语句的执行：

``` mysql
SELECT * FROM t1 JOIN (SELECT * FROM t2 WHERE data > 0) AS td WHERE td.pk=t1.pk;
```

考虑到子查询`SELECT * FROM t2`被指定为派生表`td`，所以在`create_tmp_table`函数打了断点，看是否会创建临时表。结果不出意料，并没有命中断点，看了一下代码才知道对于符合一定条件的派生表，MySQL会把对它的引用替换成对基表的引用。所以上述语句会被等价转换成类似下面的语句：

``` mysql
SELECT * FROM t1 JOIN t2 WHERE t2.data > 0 AND t1.pk = t2.pk;
```

#### PT_derived_table对象

`PT_derived_table`是一个派生表的`AST`节点对象，里面存储了派生表的信息，如别名（`MySQL`语法规定了一个派生表一定会有一个别名），构成派生表的`SELECT`语句，还有可能有用户指定的列名。

##### 生成派生表的TABLE_LIST 对象

`PT_derived_table::contextualize`成员函数将派生表封装成了一个供优化器使用的`TABLE_LIST`对象。

``` c
bool PT_derived_table::contextualize(Parse_context *pc) {

  if (m_subquery->contextualize(pc)) return true;
  /* 为派生表生成Table_ident对象，设置Table_ident::sel成员标记这个是派生表。 */
  Table_ident *ti = new (pc->thd->mem_root) Table_ident(unit);
  /* 加入到该SELECT引用的表链表，生成派生表的TABLE_LIST对象。 */
  value = pc->select->add_table_to_list(pc->thd, ti, m_table_alias, 0, TL_READ,
                                        MDL_SHARED_READ);
  /* 设置派生表的列。 */
  if (column_names.size()) value->set_derived_column_names(&column_names);
  /* 将派生表加入到join list。 */
  if (pc->select->add_joined_table(value)) return true;
  return false;
}
```

##### 派生表TABLE_LIST的处理

上面生成的`TABLE_LIST`对象在`SELECT_LEX::resolve_placeholder_tables`中会被处理。如果派生表满足下推的条件，则这个函数会调用`merge_derived()`进行下推。具体下推的条件和过程下一节再看。

#### 派生表的下推







