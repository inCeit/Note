#### SQL优化器模块常用的数据结构

##### Key_field

``` c++
/*
Key_field表示形如(column <op> val)的谓词表达式。当前的代码中op
可以是{'=', '<>', 'IS [NOT] NULL', 'arg1 IN arg2'}，val可以是
其他列或者表达式（包括常量表达式）。
Key_field用来分析某些列是否可以作为索引查找的一部分。如果'field'是某个
索引的组成列，则add_key_part()创建对应的Key_use对象，并插入到
JOIN::keyuse_arrary（函数update_ref_and_keys调用并传入）。

Key_field只用来在分析阶段用于确定候选的列来进行index ref access。
*/
struct Key_field {
  Item_field *item_field;  ///< Item representing the column
  Item *val;               ///< May be empty if diff constant
  uint level;
  uint optimize;  ///< KEY_OPTIMIZE_*
  bool eq_func;
  /**
    If true, the condition this struct represents will not be satisfied
    when val IS NULL.
    @sa Key_use::null_rejecting .
  */
  bool null_rejecting;
  bool *cond_guard;  ///< @sa Key_use::cond_guard
  uint sj_pred_no;   ///< @sa Key_use::sj_pred_no
};
```

##### Key_use

``` c++
/*
Key_use用来表示（table.column = val）形式的谓词表达式。其中column是通过成员key和key_part
来确定的（即key可以确定使用了哪个索引，key_part确定了这个列在索引中的位置）。成员val是一个常量
或者是其他表的列或多个列的表达式。如果val不是个常量，Key_use指定了一个equi-join谓词，并且它所
引用的表必须放到used_tables的表之后。
抽象来看，Key_use可以看做是一个有向箭线，源点是产生val的表，而终点是要用index lookup的表。下面
是几个例子：
1.  如果只有索引t3(c)，则下面的语句：
    SELECT * FROM t1, t2, t3
    WHERE t1.a = t3.c AND
          t2.b = t3.c;
    可以表示成下面：
     t1-- a ->- c --.
                    |
                    V
                    t3
                    ^
                    |
     t2-- b ->- c --'
     如果还存在索引t1(a)和t2(b)，则上面的equi-join graph会多两条边，即c->a和c->b：
    t1-- a ->- c --.    ,-- c -<- b --- t2
     ^             |    |               ^
     |             |    |               |
     `-- a -<- c - v    v-- c ->- b ----'
                     t3
2.   对于带表达式的语句：
    SELECT * FROM t1, t2, t3 WHERE t1.a + t2.b = t3.c;
     可以看做下面的图：
    t1-- a ---
              |
               >-- c --> t3
              |
    t2-- b ---
     
所有的Key_use对象会被有序的放到JOIN::keyuse_arrary中，sort_keyuse()函数里面给出了排序规则。
每个JOIN_TAB::keyuse执行了第一个同一个表的key_use数组元素。
*/
class Key_use {
 public:
  TABLE_LIST *table_ref;  //索引所在的表

  /**
    Value used for lookup into @c key. It may be an Item_field, a
    constant or any other expression. If @c val contains a field from
    another table, then we have a join condition, and the table(s) of
    the field(s) in @c val should be before @c table in the join plan.
  */
  Item *val;

  /**
    All tables used in @c val, that is all tables that provide bindings
    for the expression @c val. These tables must be in the plan before
    executing the equi-join described by a Key_use.
  */
  table_map used_tables;
  uint key;                  ///索引号（使用的table_ref中的哪个索引）
  uint keypart;              ///使用索引的列（一个索引可能有多个列）
  uint optimize;             ///< 0, or KEY_OPTIMIZE_*
  key_part_map keypart_map;  ///< like keypart, but as a bitmap
  ha_rows ref_table_rows;    ///< 一个key的大致记录数
  /**
    If true, the comparison this value was created from will not be
    satisfied if val has NULL 'value'.
    Not used if the index is fulltext (such index cannot be used for
    equalities).
  */
  bool null_rejecting;
  /**
    !NULL - This Key_use was created from an equality that was wrapped into
            an Item_func_trig_cond. This means the equality (and validity of
            this Key_use element) can be turned on and off. The on/off state
            is indicted by the pointed value:
              *cond_guard == true @<=@> equality condition is on
              *cond_guard == false @<=@> equality condition is off

    NULL  - Otherwise (the source equality can't be turned off)

    Not used if the index is fulltext (such index cannot be used for
    equalities).
  */
  bool *cond_guard;
  /**
     0..63    @<=@> This was created from semi-join IN-equality # sj_pred_no.
     UINT_MAX  Otherwise

     Not used if the index is fulltext (such index cannot be used for
     semijoin).

     @see get_semi_join_select_list_index()
  */
  uint sj_pred_no;

  /*
    The three members below are different from the rest of Key_use: they are
    set only by Optimize_table_order, and they change with the currently
    considered join prefix.
  */

  /**
     The key columns which are equal to expressions depending only of earlier
     tables of the current join prefix.
     This information is stored only in the first Key_use of the index.
  */
  key_part_map bound_keyparts;

  /**
     Fanout of the ref access path for this index, in the current join
     prefix.
     This information is stored only in the first Key_use of the index.
  */
  double fanout;

  /**
    Cost of the ref access path for the current join prefix, i.e. the
    cost of using ref access once multiplied by estimated number of
    partial rows from tables earlier in the join sequence.
    read_cost does NOT include cost of processing rows on the
    server side (row_evaluate_cost).

    Example: If the cost of ref access on this index is 5, and the
    estimated number of partial rows from earlier tables is 10,
    read_cost=50.

    This information is stored only in the first Key_use of the index.
  */
  double read_cost;
};
```

