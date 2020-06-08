#### STRAIGHT_JOIN

`STRAIGHT_JOIN`关键字用来指定一条SQL语句的JOIN顺序。官方文档上对它的介绍如下：

`STRAIGHT_JOIN`可以强制优化器在JOIN表时使用它们出现在`FROM`子句中的顺序。如果优化器产生的join顺序不是最优的，可以通过指定`STRAGIHT_JOIN`的方式来强制使用最优的join顺序。`straight_join`关键字不会对`const`或`system`类型的表产生影响。`const`和`system`类型的表指的是表最多只会有一行数据会被返回，在优化器运行时，这些行会被读出来，并用它的值替换当前引用这一行数据的地方。这些表在使用`EXPLAIN`语句时会被首先显示出来（`outer join`例外）。

`straight_join`关键字也可以放在`table_reference`的位置，在执行的时候左边的表总会比右边的表先读取。

##### straight_join在代码中对优化器的影响

下面的代码基于`MySQL 8.0.20`。

当指定`straight_join`关键字后，优化器会跳过对join order的优化。具体来说，当`Optimize_table_order::choose_table_order()`执行时会直接调用`optimize_straight_join()`，而不会调用`greedy_search()`。`optimize_straight_join()`函数不再对join顺序进行优化，而是逐个计算每个表的最优访问路径和代价。

下面是`optimize_straight_join()`函数的简化代码：

``` c++
void Optimize_table_order::optimize_straight_join(table_map join_tables)
{
  uint idx= join->const_tables;
  double rowcount= 1.0;
  double cost= 0.0;
  const Cost_model_server *const cost_model= join->cost_model();

  Opt_trace_context * const trace= &join->thd->opt_trace;
  for (JOIN_TAB **pos= join->best_ref + idx; *pos; idx++, pos++)
  {
    JOIN_TAB *const s= *pos;
    POSITION *const position= join->positions + idx;

    /* 根据当前部分优化的结果，获取s的最佳访问路径 */
    best_access_path(s, join_tables, idx, false, rowcount, position);

    // 计算加上s之后，当前执行计划的代价。
    position->set_prefix_join_cost(idx, cost_model);

    /* rowcount是目前执行计划可能读取的总行数，cost是对应的总代价。 */
    rowcount= position->prefix_rowcount;
    cost=     position->prefix_cost;

	/* 将优化后的表从待优化的表集合中移除。 */
    join_tables&= ~(s->table_ref->map());
  }

  /* 如果需要临时表排序，则要把总代价加上所要读取的总行数。 */
  if (join->sort_by_table &&
      join->sort_by_table != join->positions[join->const_tables].table->table())
    cost+= rowcount;  // We have to make a temp table

  /* 将当前的计划放入到最优的计划中。 */
  memcpy(join->best_positions, join->positions, sizeof(POSITION)*idx);

  join->best_read= cost - 0.001;
  join->best_rowcount= (ha_rows)rowcount;
}
```

上面的函数可以看出，`optimize_straight_join`并不对join order进行优化，而是按照表原始的先后顺序，逐个计算表的最优访问路径和代价。用来根据当前的join结果来计算下一个表的最优访问路径的函数是`best_access_path`。



``` c++

```





