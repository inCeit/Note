## InnoDB中的行格式

​		本文档是关于`InnoDB`中行格式的文档阅读笔记和源码阅读笔记。本文中“字段”和“列”可以相互替换。最后，看一下索引页的格式。

##### 需求来源

​		物理复制中需要解决物理日志数据到数据页数据的回放，这其中需要了解数据的物理存放格式。另外，阅读B+树分裂和合并的代码也需要了解关于行格式的信息。

### 1. InnoDB Row Formats 简介

​		本部分来自于官方文档的摘抄：https://dev.mysql.com/doc/refman/5.7/en/innodb-row-format.html#innodb-row-format-defining

​		一个表的行格式决定了表中的数据在物理上如何存储的，同时行的物理存储方式也影响着查询的性能和DML操作。显然，如果一个页上存储的行数越多，索引查找的速度就会越快，因为所需的内存和I/O次数变少了。

​		在物理上，聚簇索引包含了一个表的所有列，一个二级索引包含了一个表的索引列和主键列。变长列（`Variable-length columns`）是一个例外，如果一个变长列太长而不能放入一个B+树页，则会放入到溢出页（`overflow page`）中，这种列称为`off-page columns`。`off-page columns`的值存储在一个由溢出页构成的单向列表上。视列的长度而定，一个变长列可能全部放入一个B+树页，或只把列的前缀放入B+树页，这样一来可以节省空间，二来可以避免访问溢出页。

​		`InnoDB`存储引擎目前支持四种行格式：`REDUNANT`,`COMPACT`,`DYNAMIC`和`COMPRESSED`。下面一一介绍这四种行格式。

#### 1.1 REDUNDANT Row Format

​		`REDUNDANT` 行格式支持老版本的MySQL。`Barracuda`和`Antelope`两种文件格式都支持`REDUNDANT`行格式。

​		`REDUNDANT`行格式将一个变长列（`VARCHAR`,`VARBINARY`,`BLOB` & `TEXT`）的前768个字节存放到B+树的索引页上，剩余的内容放在溢出页上。另外，定长列如果大于768字节，会被转换为变长列，存放在溢出页上。例如一个列类型为`CHAR(255)`，在每个字符占有3个以上字节的情况下，该列可能超过768字节。

​		如果一个列占768字节或更少，则不会使用溢出页，会被完整地存放在B+树的索引页上。

##### REDUNDANT Row Format Storage Characteristics

​		`REDUNDANT`行格式有以下存储特性：

- 6-byte 的头部：用于连接连续的记录和实现行级别的锁。
- 聚簇索引记录包含所有的用户定义的列，除此之外，还有两个列，分别是6-byte事务ID列和7-byte的回滚指针列。
- 如果用户没有定义主键，则每条聚簇索引记录还包含6-byte的row id列。
- 每条二级索引记录包含所有未出现在二级索引列上的主键列。
- 每条记录都包含了指向记录中**每个字段**的指针，如果**记录**的长度小于128-byte（注意这里是记录的总长度，而不是每个字段的长度），则指针大小为1个字节，否则使用2个字节。这个指针数组称为record directory，指针指向的区域是记录的实际数据部分，称为record data。

- 在内部，定长字符列（例如CHAR（10））以定长格式存储。 尾随空格不会从VARCHAR列中截断。
- 定长列如果大于768字节，会被转换为变长列，存放在溢出页上。例如一个列类型为`CHAR(255)`，在每个字符占有3个以上字节的情况下，该列可能超过768字节。
- 对于一个SQL `NULL`值，`REDUNDANT`行格式会在record directory预留1或2个byte。另外，对于一个类型为变长列的SQL `NULL`值， `REDUNDANT`行格式会在record data中预留0-byte。对于，定长列，会在record data中预留固定长度的字节数。通过预留空间，将来该`NULL`字段更新为非`NULL`值时可以实现原地更新，而不会引起页面分裂。

#### 1.2 COMPACT Row Format

​		`COMPACT`行格式较`REDUNDANT`行·格式会节约大约20%的存储空间，代价是会提升`CPU`使用率。`Antelope`和`Barracuda`两种文件格式都支持`COMPACT`行格式。

​		`COMPACT`行格式将一个变长列（`VARCHAR`,`VARBINARY`,`BLOB` & `TEXT`）的前768个字节存放到B+树的索引页上，剩余的内容放在溢出页上。另外，定长列如果大于768字节，会被转换为变长列，存放在溢出页上。例如一个列类型为`CHAR(255)`，在每个字符占有3个以上字节的情况下，该列可能超过768字节。

​		如果一个列占768字节或更少，则不会使用溢出页，会被完整地存放在B+树的索引页上。

##### COMPACT Row Format Storage Characteristics

​		`COMPACT`行格式有以下存储特性：

- 每条记录包含5-byte的头部（fixed-length header），在头部之前还可能存在一个变长的头部（variable-length header）：记录头部用于连接连续的用户记录，和实现行锁。
- variable-length header包含一个bit vector，用来标记记录值为`NULL`的列。例如，一个索引中有N列的值可以为NULL，则这个bit vector大小为`CELING(N/8)`个字节。值为`NULL`的列不再占用任何空间，只会在bit vector的对应bit上做标记。variable-length header还包含了变长列的长度，每个变长列的长度值占用1或2个字节，这取决于列的最大长度。当所有的列都是定长且非NULL时，记录不会有variable-length header。
- 对于每个`non-NULL`的变长字段，variable-length header使用1~2byte/字段来保存字段的长度。2byte的情况只用于下面两种情况：1）字段的部分内容存放在溢出页上；2）最大长度超过255byte，并且实际长度超过127个字节。对于存在溢出页的字段，这两个字节指示了字段存放在B+树索引页内的长度+20字节的外部存储指针。因为索引页内的长度为768字节，所以总长度为768+20。20字节的指针存放着字段的实际长度。
- record header后面就是记录`non-NULL`的列数据。
- 聚簇索引记录包含所有的用户定义的列，除此之外，还有两个列，分别是6-byte事务ID列和7-byte的回滚指针列。
- 如果用户没有定义主键，则每条聚簇索引记录还包含6-byte的row id列。
- 每条二级索引记录包含所有未出现在二级索引列上的主键列。
- 所有定长字符集、字长字符列，例如`char(10)`，都会采用定长列。另外，`varchar`列不会进行尾部空格的截断。
- 定长列如果大于768字节，会被转换为变长列，存放在溢出页上。例如一个列类型为`CHAR(255)`，在每个字符占有3个以上字节的情况下，该列可能超过768字节。
- 对于变长字符集，比如`utf8mb`和`utf8mb4`，`InnoDB`在尝试存储`CHAR(N)`类型的字段时会截断至刚好能存放列的长度。我们不妨设某个字符集`CHAR(N)`最少需要`min(N)`个字节，最多需要`max(N)`个字节，而对于一个特定的字段值来说，其真实长度为`real(N)`。则在`COMPACT`行格式下，`InnoDB`会用`real(N)+min(N)`个字节来存放这个字段，即使用`real(N)`个字节来存放字段的值，并预留`min(N)`个字节来满足未来修改的需要。相比之下，`REDUNDANT`行格式会使用`max(N)`来存放这个字段，而不管这个字段值的真实大小。

#### 1.3 DYNAMIC Row Format

​		`DYNAMIC`行格式具有和`COMPACT`相同的存储属性，并增强了一些存储属性来支持变长列和大索引值前缀。

​		只有`Barracuda`文件格式支持`DYNAMIC`行格式。

​		当以`ROW_FORMAT=DYNAMIC`选项创建一个表时，`InnoDB`**可能会**将大的变长列（例如`VARCHAR`, `VARBINARY` , `BLOB`和`TEXT`）完全的放在溢出页上，聚簇索引记录上只包含20byte的指针，指向溢出页上的记录。定长列如果大于768字节，会被转换为变长列，存放在溢出页上。例如一个列类型为`CHAR(255)`，在每个字符占有3个以上字节的情况下，该列可能超过768字节。

​		在`DYNAMIC`行格式下，一个列是否会被放入到溢出页取决于页面大小和行的总大小。当一行记录太大时，最长的N个列会被选中放入到溢出页中，直到聚簇索引记录的大小合适于一个B+树页面。另外，`TEXT`和`BLOB`字段的长度如果小于等于40bytes则不会放入溢出页。

​		`DYNAMIC`行格式在记录的总长度比较小时会把整行都放入到索引页，而不管其中某个列的长度。同时在记录过大时，`DYNAMIC`行格式的处理方式是基于下面的思想：如果一个字段太大，则把一部分放入溢出页，反而不如把整个字段都放入溢出页效率更高。

​		`DYNAMICA`行格式最大支持3072个字节的前缀索引（`REDUNDANT`和`COMPACT`行格式只支持最大767字节的前缀索引），这个功能可以通过`innodb_large_prefix`来进行控制，默认情况下是开启的。

##### DYNAMIC Row Format Storage Characteristics

​		`DYNAMIC`行格式是`COMPACT`行格式的一个变种，有着与`COMPACT`行格式相同的存储特性。

#### 1.4 COMPRESSED Row Format

​		`COMPRESSED`行格式提供了与`DYNAMIC`行格式相同的功能和存储特性，但是额外提供了表和索引数据的压缩功能。

​		只有`Barracuda`文件格式支持`COMPRESSED`行格式，`Antelope`不支持。

​		`COMPRESSED`行格式在溢出页存储方面与`DYNAMIC`行格式相似，但是使用更小的数据页尺寸来满足压缩特性。在`COMPRESSED`行格式中，`KEY_BLOCK_SIZE`选项控制着多少列数据可以存放在聚簇索引上，多少的列数据可以放在溢出页上。

​		`COMPRESSED`行格式最大支持3072个字节的前缀索引（`REDUNDANT`和`COMPACT`行格式只支持最大767字节的前缀索引），这个功能可以通过`innodb_large_prefix`来进行控制，默认情况下是开启的。

​		使用`COMPRESSED`行格式的表可以放在`file-per-table Tablespace`或`General Tablespace`。系统表空间不支持`COMPRESSED`行格式。如果要存储一个`COMPRESSED`格式的表在`file-per-table Tablespace`，则`innodb_file_per_table`选项必须打开，`innodb_file_format`选项必须设置成`Barracuda`。另外，`innodb_file_per_table`和`innodb_file_format`两个选项对`General Tablespace`无效。`General Tablespace`支持所有的行格式，但是不支持同时存放`COMPRESSED`的表和非`COMPRESSED`的表，因为`COMPRESSED`和非`COMPRESSED`的表的数据页物理大小可能不一致。

#### 1.5 四种行格式特性对比

![1573697210571](C:\Users\\AppData\Roaming\Typora\typora-user-images\1573697210571.png)

#### 1.6 MySQL中关于行和列的限制

##### 列数限制

MySQL支持的最大列数为4096，但是实际支持的行数还受下面的几个条件限制：

- 行大小的限制，因为一条记录中的所有列加在一起不能超过行支持的最大尺寸；
- 存储引擎可能对支持的行数有额外的限制，例如`InnoDB`会限制一个表的最列数为1017；

##### 行大小限制

MySQL可以支持的最大的行大小为65535个字节，即使存储引擎可以支持更大的行。`BLOB`和`TEXT`的内容不会算入这个大小，因为它们可以放入到溢出页中。此外，行大小还有其他限制：

- 对于`InnoDB`表，行的大小要略小于页大小的一半。例如对于16K的`InnoDB`表，行的大小会略小于8K，对于64K的页，行的大小会略小于16K（注意这里不是计算错了，从2.2节的rec_convert_dtuple_to_rec_comp()函数中可以看到在物理上记录的每个变长字段都会使用1~2字节来存储字段在记录内的偏移值，如果该偏移值存在2个字节中，并且这2个字节的最高的2位被作为特殊标志，所以能表示的最大大小是16384）。注意，这里的大小也是不计算`TEXT`和`BLOB`字段大小的，因为会被放入到溢出页中。

### 2. 行格式实现

​		本节介绍`InnoDB`代码中记录如何按照一定的行格式存入到数据页上。

#### 2.1 行的存储结构

​		由第1节的内容可知，除了`REDUNDANT`，其他三种行格式都有着相同的存储结构。所以无论是代码还是注释中，都把`REDUNDANT`行格式记为`OLD`的格式，而`COMPACT`、`DYNAMIC`和`COMPRESSED`三种行格式统一记为`NEW`的格式，并习惯地使用`compact**`的名字。

​		下图显示了两类行格式的存储结构：

![1573713355773](C:\Users\\AppData\Roaming\Typora\typora-user-images\1573713355773.png)

​		上图中，除了`Record Header`字段，其他字段已经在第1节中介绍了，这里看一下`Record Header`字段内容：

- next rec ：指向下一条记录在页面内的偏移值，因为索引页内的记录在逻辑上是一个单链表，用此字段连接。

- heap no : 记录在页面内的id，一个索引页中所有的记录都有一个唯一的heap no（即使已经打了delete mark的记录也有唯一的heap no）。heap no用于实现行锁时标识一条记录。

- n_owned ：记录当前所在的directory slot里有几条记录。

- info bits ：记录信息标志位，目前有两位使用：

  - REC_INFO_MIN_REC_FLAG（0x10）—— 标识该记录是中间节点上最左的一条记录（FIL_PAGE_PREV是FIL_NULL，PAGE_LEVEL不为0）；
  - REC_INFO_DELETED_FLAG（0x20）——标识该记录已经被打上delete标记。

  对于新记录格式，还有以下字段：

- status ：记录的“状态”，总共3位，有5个取值：

  - REC_STATUS_ORDINARY（00） — 叶子节点记录；
  - REC_STATUS_NODE_PTR（001） — 中间节点记录；
  - REC_STATUS_INFIMUM （010）— infimum记录；
  - REC_STATUS_SUPREMUM （011） — supremum记录；
  - 1xx — 保留状态，未使用。

  对于老记录，还有以下字段：

- short flag：如果没有外部字段，且记录总大小小于等于REC_1BYTE_OFFS_LIMIT（0x7F），则设置此flag；

- number of fields：记录的总字段数。

##### 2.1.1 行信息读取的函数 — rec_get_offsets_func()

​		在`InnoDB`的代码中，如果读取一条记录时，通常第一步是调用`rec_get_offsets()`函数来获取每个字段的值在记录内的偏移值，再通过这些偏移值读取每个字段在记录内的数据。`rec_get_offsets()`是一个宏定义，实际上调用的是`rec_get_offsets_func()`，下面给出了这个函数的注解版源码：

``` c++
rec_get_offsets_func(rec, index, offsets) {
    ulint n;	//记录的总字段数
    /* 1. 首先获取记录中字段的数量 */
    /* 如果是新记录格式，则记录总字段数从索引中获取。 */
    if (dict_table_is_comp(index->table)) {
        switch(rec_get_status(rec)) {
            /* 如果是叶子节点上的记录，则获取聚簇索引的总字段数。 */
            case REC_STATUS_ORDINARY:
                n = dict_index_get_n_fields(index);
            /* 如果是中间节点，除了要获取可以唯一确定索引记录的字段数外，
            还需要加上孩子节点页号字段 */
            case REC_STATUS_NODE_PTR:
                n = dict_index_get_n_unique_in_tree_nonleaf(index) + 1;
            case REC_STATUS_INFIMUM:
            case REC_STATUS_SUPREMUM:
                /* infimum和supremum特殊记录只有一个字段 */
                n = 1;
        }
    } else {
        /* 如果是旧记录格式，则从记录头部中的number of fields（见2.1节）部分直接读取。*/
        n = rec_get_n_fields_old(rec);
    }
    /* n是字段偏移值的数量，1是用来存放extra_size，REC_OFFS_HEADER_SIZE在非DEBUG模式下
    占2个字节，分别存放offsets数组的大小，和记录字段的数量。*/
    size = n + (1 + REC_OFFS_HEADER_SIZE);
    /* 2. 针对每个字段，计算出每个字段的数据在记录中的偏移值。 */
    rec_init_offsets(rec, index, offsets);
    return(offsets);
}
```

​		`rec_init_offsets(rec, index, offsets)`函数会读取每个字段的长度值，并计算每个字段在记录中的**偏移值**。`offsets`数组在代码中经常会被用到，它的结构如下所示：

![1573782166854](C:\Users\\AppData\Roaming\Typora\typora-user-images\1573782166854.png)

#### 2.2 如何把内存数据转化为物理记录

​		在`InnoDB`中，一条记录在内存中使用一个`dtuple_t`对象来保存，被称为“逻辑记录”。要把一条逻辑记录转换成可以存放在数据页上的物理记录需要通过`rec_convert_dtuple_to_rec()`函数来完成。这个函数根据`dtuple_t`对象中各个字段的值来构造record header和record directory等（即2.1.1中提到的extra size的部分）。函数的注解版代码如下所示：

``` c++
rec_t*
rec_convert_dtuple_to_rec(
/*======================*/
	byte*			buf,	/*!< in: start address of the
					physical record */
	const dict_index_t*	index,	/*!< in: record descriptor */
	const dtuple_t*		dtuple,	/*!< in: data tuple */
	ulint			n_ext)	/*!< in: number of
					externally stored columns */
{
    /* 如果是compact行格式（包含compact, dynamic和compressed，这三种行格式
    的存储属性相同，所以就用compact来表示）。 */
	if (dict_table_is_comp(index->table)) {
		rec = rec_convert_dtuple_to_rec_new(buf, index, dtuple);
	} else {
        /* 如果是redundant行格式 */
		rec = rec_convert_dtuple_to_rec_old(buf, dtuple, n_ext);
	}
	return(rec);
}

// ## 1. compact行格式的转换函数rec_convert_dtuple_to_rec_new()；
rec_convert_dtuple_to_rec_new(
/*==========================*/
	byte*			buf,	/*!< in: start address of
					the physical record */
	const dict_index_t*	index,	/*!< in: record descriptor */
	const dtuple_t*		dtuple)	/*!< in: data tuple */
{

	status = dtuple_get_info_bits(dtuple) & REC_NEW_STATUS_MASK;
	/* 1. 获取转换成物理记录时记录的尺寸，其中函数返回值是最终物理记录的总大小，而
    extra_size返回记录的record header等不包含用户数据部分的大小。*/
    rec_get_converted_size_comp(
		index, status, dtuple->fields, dtuple->n_fields, &extra_size);
	rec = buf + extra_size;
	
    /* 2.将用户数据转换成物理记录 */
	rec_convert_dtuple_to_rec_comp(
		rec, index, dtuple->fields, dtuple->n_fields, NULL,
		status, false);

	/* Set the info bits of the record */
	rec_set_info_and_status_bits(rec, dtuple_get_info_bits(dtuple));

	return(rec);
}
// ### 1.1 compact行格式的转换函数 rec_convert_dtuple_to_rec_comp()
void
rec_convert_dtuple_to_rec_comp()
{
    //1. 确定中间节点字段个数
	if (temp) {
		n_node_ptr_field = ULINT_UNDEFINED;
		nulls = rec - 1;
		if (dict_table_is_comp(index->table)) {
			temp = false;
		}
	} else {
		nulls = rec - (REC_N_NEW_EXTRA_BYTES + 1);
		switch (UNIV_EXPECT(status, REC_STATUS_ORDINARY)) {
		case REC_STATUS_ORDINARY:
			n_node_ptr_field = ULINT_UNDEFINED;
			break;
		case REC_STATUS_NODE_PTR:
			n_node_ptr_field = n_fields - 1;
			break;
		case REC_STATUS_INFIMUM:
		case REC_STATUS_SUPREMUM:
			n_node_ptr_field = ULINT_UNDEFINED;
			break;
		}
	}
	end = rec;
	
    //2. reset NULLL Bit Vector，用来标记值为NULL的列
	if (n_fields != 0) {
		n_null = index->n_nullable;
		lens = nulls - UT_BITS_IN_BYTES(n_null);
		memset(lens + 1, 0, nulls - lens);
	}

	/* 遍历记录每个字段，分别存储字段数据和字段长度 */
	for (i = 0; i < n_fields; i++) {
		dict_col_t*		col = NULL;

		field = &fields[i];

		type = dfield_get_type(field);
		len = dfield_get_len(field);

        //如果要处理的记录是中间节点记录，且到了最后一个，则加上4个字节（孩子节点页号）
		if (UNIV_UNLIKELY(i == n_node_ptr_field)) {
			memcpy(end, dfield_get_data(field), len);
			end += REC_NODE_PTR_SIZE;
			break;
		}

		if (!(dtype_get_prtype(type) & DATA_NOT_NULL)) {
			/* 如果记录为NULL，则在Bit Vector中标记对应的位 */
			if (dfield_is_null(field)) {
				*nulls |= null_mask;
				null_mask <<= 1;
				continue;
			}
		}

		ifield = dict_index_get_nth_field(index, i);
		fixed_len = ifield->fixed_len;
		col = ifield->col;

        if (fixed_len) {
            /* 如果字段有固定长度，则不会在头部记录其长度 */
        } else if (dfield_is_ext(field)) {
            /* 对于外部存储的字段统一使用2个字节表示长度，
            注意：0xc0（1100 0000）表示：
            1）最高位的1表示这个长度字段占两个字节；
            2）次高位的1表示这个字段是外部字段，使用了溢出页；
            */
			*lens-- = (byte) (len >> 8) | 0xc0;
			*lens-- = (byte) len;
		} else {
			/* 如果字段数据小于128字节时，则使用一个字节表示长度 */
			if (len < 128 || !DATA_BIG_LEN_MTYPE(
				dtype_get_len(type), dtype_get_mtype(type))) {
				*lens-- = (byte) len;
			} else {
				ut_ad(len < 16384);
                /* 当字段大于等于128字节时使用2个字节表示长度。 
                注意：0x80（1000 0000）表示：
            	1）最高位的1表示这个长度字段占两个字节；
            	*/
				*lens-- = (byte) (len >> 8) | 0x80;
				*lens-- = (byte) len;
			}
		}

		memcpy(end, dfield_get_data(field), len);
		end += len;
	}

	if (!num_v) {
		return;
	}
}
```

##### 2.2.1 对于外部字段的处理

​		外部字段是指字段的部分数据或全部数据放在溢出页上的字段。外部字段一般都是`varchar`,`text`,`blob`等类型，这部分的内容将在第3节数据表达结束后给出。

### 3. InnoDB中的数据表达

​		本节介绍`InnoDB`中数据表达，这包括`InnoDB`中不同类型的数据在物理上的存储方式，`InnoDB`对字符集的支持方式，MySQL数据类型到`InnoDB`数据类型的转换。数据类型是行格式的重要组成部分，所以本节介绍此部分的内容。

#### 3.1 基础数据类型的内存表示

​		`InnoDB`中基础数据类型使用`dtype_t`(row554@data0type.h)来表示，每个列都是一个基础数据类型。`dtype_t`的定义如下：

``` c
struct dtype_t{
	unsigned	prtype:32;	/*!< precise type/ 精确类型
					包括：MySQL数据类型，字符集代码，是否为NULL，
					是否带符号，是否是binary string，是否是
					true VARCHAR（会有专门的长度字段）。 */
	unsigned	mtype:8;	/*!< main data type/ 主数据类型 */

	unsigned	len:16;		/* 数据长度，值视类型而定 */
	unsigned	mbminmaxlen:5;	/* 字符集中每个字符的可能的最大字节和最小字节. */
};
```

​		在`dtype_t`中，数据类型分为了`mtype`和`ptype`，即主类型和精确类型。主类型和精确类型两个值确定了字节排序的方式。下面看一下主类型和精确类型的可能取值。

##### 3.1.1 主类型与精确类型的取值

​		主类型说明了一个值是数值还是字符串，指定了精确类型可以取的值。主类型的取值目前有16个，如下表所示：

![1573816633558](C:\Users\\AppData\Roaming\Typora\typora-user-images\1573816633558.png)

#### 3.2 其他数据结构

3.1节介绍的`dtype_t`算是`InnoDB`里最底层的数据类型。为了表示不同的数据对象，在`dtype_t`之上，还有`dfield_t`来表示一个字段，以及`dtuple`来表示一条记录。显然他们的关系如下：

```c++
dtuple_t _____ dfield_t -> dtype_t
         |____ dfield_t -> dtype_t
         | ...多个字段的列表...
         |____ dfield_t -> dtype_t
```

`dfield_t`和`dtuple_t`的定义如下：

``` c++
#define UNIV_SQL_NULL ULINT32_UNDEFINED

/** Structure for an SQL data field */
struct dfield_t{
	void*		data;	/*!< pointer to data */
	unsigned	ext:1;	/*!< TRUE=externally stored, FALSE=local */
	unsigned	spatial_status:2;
				/*!< spatial status of externally stored field
				in undo log for purge */
	unsigned	len;	/*!< data length; UNIV_SQL_NULL if SQL null */
	dtype_t		type;	/*!< type of data */
};

/** Structure for an SQL data tuple of fields (logical record) */
struct dtuple_t {
	ulint		info_bits;	/*!< info bits of an index record:
					the default is 0; this field is used
					if an index record is built from
					a data tuple */
	ulint		n_fields;	/*!< number of fields in dtuple */
	ulint		n_fields_cmp;	/*!< number of fields which should
					be used in comparison services
					of rem0cmp.*; the index search
					is performed by comparing only these
					fields, others are ignored; the
					default value in dtuple creation is
					the same value as n_fields */
	dfield_t*	fields;		/*!< fields */
	ulint		n_v_fields;	/*!< number of virtual fields */
	dfield_t*	v_fields;	/*!< fields on virtual column */
	UT_LIST_NODE_T(dtuple_t) tuple_list;
					/*!< data tuples can be linked into a
					list using this field */
};
```

无论是`dtuple_t`还是`dfield_t`各成员定义都清晰明了，这里不一一解释了。

### 4. 索引页结构













 
