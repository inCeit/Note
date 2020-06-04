### JOIN PLAN生成中用到的函数

`JOIN::make_join_plan()`成员函数用于生成`JOIN PLAN`。这里主要记录该成员函数调用的函数。

##### JOIN::init_planner_arrays()

`JOIN_TAB`内存对象的生成。

``` c
bool JOIN::init_planner_arrays() {
  // Up to one extra slot per semi-join nest is needed (if materialized)
  const uint sj_nests = select_lex->sj_nests.size();
  const uint table_count = select_lex->leaf_table_count;
  /* 根据表的数量生成JOIN_TAB和QEP_shared的对象，并把QEP_shared赋给JOIN_TABLE。 */
  if (!(join_tab = alloc_jtab_array(thd, table_count))) return true;

  /*
    We add 2 cells:
    - because planning stage uses 0-termination so needs +1
    - because after get_best_combination, we don't use 0-termination but
    need +2, to host at most 2 tmp sort/group/distinct tables.
  */
  if (!(best_ref = (JOIN_TAB **)thd->alloc(
            sizeof(JOIN_TAB *) *
            (table_count + sj_nests + 2 + m_windows.elements))))
    return true;

  // sort/group tmp tables have no map
  if (!(map2table = (JOIN_TAB **)thd->alloc(sizeof(JOIN_TAB *) *
                                            (table_count + sj_nests))))
    return true;

  if (!(positions = new (thd->mem_root) POSITION[table_count])) return true;

  if (!(best_positions = new (thd->mem_root) POSITION[table_count + sj_nests]))
    return true;

  /*
    Initialize data structures for tables to be joined.
    Initialize dependencies between tables.
  */
  JOIN_TAB **best_ref_p = best_ref;
  TABLE_LIST *tl = select_lex->leaf_tables;

  for (JOIN_TAB *tab = join_tab; tl; tab++, tl = tl->next_leaf, best_ref_p++) {
    *best_ref_p = tab;
    TABLE *const table = tl->table;
    tab->table_ref = tl;
    tab->set_table(table);
    /* 这里会从handler层获取表的大致行数。 */
    const int err = tl->fetch_number_of_rows();

    // Initialize the cost model for the table
    table->init_cost_model(cost_model());

    if (err) {
      table->file->print_error(err, MYF(0));
      return true;
    }
    all_table_map |= tl->map();
    tab->set_join(this);
    /* 设置JOIN_TAB的依赖关系。问题是t1的依赖关系如何确定的。 */
    tab->dependent = tl->dep_tables;  // Initialize table dependencies
    table->quick_condition_rows = table->file->stats.records;

    /* 设置JOIN的条件。 */
    tab->init_join_cond_ref(tl);
    tables++;  // Count number of initialized tables
  }

  primary_tables = tables;
  *best_ref_p = nullptr;  // Last element of array must be NULL

  return false;
}
```

##### update_ref_and_keys()

根据`WHERE`或`OUT JOIN`条件，来判断每个表应该使用的索引。

``` c++
/**
  Update keyuse array with all possible keys we can use to fetch rows.

  @param       thd            session context
  @param[out]  keyuse         Put here ordered array of Key_use structures
  @param       join_tab       Array in table number order
  @param       tables         Number of tables in join
  @param       cond           WHERE condition (note that the function analyzes
                              join_tab[i]->join_cond() too)
  @param       normal_tables  Tables not inner w.r.t some outer join (ones
                              for which we can make ref access based the WHERE
                              clause)
  @param       select_lex     current SELECT
  @param[out]  sargables      Array of found sargable candidates

  @returns false if success, true if error
*/

static bool update_ref_and_keys(THD *thd, Key_use_array *keyuse,
                                JOIN_TAB *join_tab, uint tables, Item *cond,
                                table_map normal_tables, SELECT_LEX *select_lex,
                                SARGABLE_PARAM **sargables) {
  DBUG_ASSERT(cond == nullptr || cond->is_bool_func());
  uint and_level, i, found_eq_constant;
  Key_field *key_fields, *end, *field;
  size_t sz;
  uint m = max(select_lex->max_equal_elems, 1U);
  JOIN *const join = select_lex->join;
  /*
    We use the same piece of memory to store both  Key_field
    and SARGABLE_PARAM structure.
    Key_field values are placed at the beginning this memory
    while  SARGABLE_PARAM values are put at the end.
    All predicates that are used to fill arrays of Key_field
    and SARGABLE_PARAM structures have at most 2 arguments
    except BETWEEN predicates that have 3 arguments and
    IN predicates.
    This any predicate if it's not BETWEEN/IN can be used
    directly to fill at most 2 array elements, either of Key_field
    or SARGABLE_PARAM type. For a BETWEEN predicate 3 elements
    can be filled as this predicate is considered as
    saragable with respect to each of its argument.
    An IN predicate can require at most 1 element as currently
    it is considered as sargable only for its first argument.
    Multiple equality can add  elements that are filled after
    substitution of field arguments by equal fields. There
    can be not more than select_lex->max_equal_elems such
    substitutions.
  */
  sz = max(sizeof(Key_field), sizeof(SARGABLE_PARAM)) *
       (((select_lex->cond_count + 1) * 2 + select_lex->between_count) * m + 1);
  if (!(key_fields = (Key_field *)thd->alloc(sz)))
    return true; /* purecov: inspected */
  and_level = 0;
  field = end = key_fields;
  *sargables = (SARGABLE_PARAM *)key_fields +
               (sz - sizeof((*sargables)[0].field)) / sizeof(SARGABLE_PARAM);
  /* set a barrier for the array of SARGABLE_PARAM */
  (*sargables)[0].field = nullptr;

  if (cond) {
    if (add_key_fields(thd, join, &end, &and_level, cond, normal_tables,
                       sargables))
      return true;
    for (Key_field *fld = field; fld != end; fld++) {
      /* Mark that we can optimize LEFT JOIN */
      if (fld->val->type() == Item::NULL_ITEM &&
          !fld->item_field->field->is_nullable()) {
        /*
          Example:
          SELECT * FROM t1 LEFT JOIN t2 ON t1.a=t2.a WHERE t2.a IS NULL;
          this just wants rows of t1 where t1.a does not exist in t2.
        */
        fld->item_field->field->table->reginfo.not_exists_optimize = true;
      }
    }
  }

  for (i = 0; i < tables; i++) {
    /*
      Block the creation of keys for inner tables of outer joins.
      Here only the outer joins that can not be converted to
      inner joins are left and all nests that can be eliminated
      are flattened.
      In the future when we introduce conditional accesses
      for inner tables in outer joins these keys will be taken
      into account as well.
    */
    if (join_tab[i].join_cond()) {
      if (add_key_fields(thd, join, &end, &and_level, join_tab[i].join_cond(),
                         join_tab[i].table_ref->map(), sargables))
        return true;
    }
  }

  /* Process ON conditions for the nested joins */
  for (TABLE_LIST *tl : select_lex->top_join_list) {
    if (tl->nested_join &&
        add_key_fields_for_nj(thd, join, tl, &end, &and_level, sargables))
      return true;
  }

  /* Generate keys descriptions for derived tables */
  if (select_lex->materialized_derived_table_count ||
      select_lex->table_func_count) {
    if (join->generate_derived_keys()) return true;
  }
  /* fill keyuse with found key parts */
  for (; field != end; field++) {
    if (add_key_part(keyuse, field)) return true;
  }

  if (select_lex->ftfunc_list->elements) {
    if (add_ft_keys(keyuse, join_tab, cond, normal_tables, true)) return true;
  }

  /*
    Sort the array of possible keys and remove the following key parts:
    - ref if there is a keypart which is a ref and a const.
      (e.g. if there is a key(a,b) and the clause is a=3 and b=7 and b=t2.d,
      then we skip the key part corresponding to b=t2.d)
    - keyparts without previous keyparts
      (e.g. if there is a key(a,b,c) but only b < 5 (or a=2 and c < 3) is
      used in the query, we drop the partial key parts from consideration).
    Special treatment for ft-keys.
  */
  if (!keyuse->empty()) {
    Key_use *save_pos, *use;

    std::sort(keyuse->begin(), keyuse->begin() + keyuse->size(), sort_keyuse);

    const Key_use key_end(nullptr, nullptr, 0, 0, 0, 0, 0, 0, false, nullptr,
                          0);
    if (keyuse->push_back(key_end))  // added for easy testing
      return true;

    use = save_pos = keyuse->begin();
    const Key_use *prev = &key_end;
    found_eq_constant = 0;
    for (i = 0; i < keyuse->size() - 1; i++, use++) {
      TABLE *const table = use->table_ref->table;
      if (!use->used_tables && use->optimize != KEY_OPTIMIZE_REF_OR_NULL)
        table->const_key_parts[use->key] |= use->keypart_map;
      if (use->keypart != FT_KEYPART) {
        if (use->key == prev->key && use->table_ref == prev->table_ref) {
          if (prev->keypart + 1 < use->keypart ||
              (prev->keypart == use->keypart && found_eq_constant))
            continue;                  /* remove */
        } else if (use->keypart != 0)  // First found must be 0
          continue;
      }

      /*
        Protect against self assignment.
        The compiler *may* generate a call to memcpy() to do the assignment,
        and that is undefined behaviour (memory overlap).
       */
      if (save_pos != use) *save_pos = *use;
      prev = use;
      found_eq_constant = !use->used_tables;
      /* Save ptr to first use */
      if (!table->reginfo.join_tab->keyuse())
        table->reginfo.join_tab->set_keyuse(save_pos);
      table->reginfo.join_tab->checked_keys.set_bit(use->key);
      save_pos++;
    }
    i = (uint)(save_pos - keyuse->begin());
    keyuse->at(i) = key_end;
    keyuse->chop(i);
  }
  print_keyuse_array(&thd->opt_trace, keyuse);
  /*
    Number of functions here call val_x() methods, which might throw an error.
    Catch those errors here.
  */
  return thd->is_error();
}
```

