### SQL逻辑优化——条件变换

逻辑优化是的MySQL优化器执行的第一步，而条件变换又是逻辑优化的第一步。条件变换是对SQL语句中的`where`和`join`条件进行等价变换，方便后续优化使用。

MySQL的条件变换会做以下几部分等价变化：

- Negation elimination

> 对条件进行NOT转换，例如对条件：NOT (t1.a > 0 OR t1.b < 100) 会被等价变换为 (t1.a <= 0 AND t1.b >= 100)。

- Equality propagations

> 对多个等式条件进行合并，例如：t1.a = t2.b AND t2.b = t3.c 会被转换为 t1.a = t2.b = t3.c，这样转换后JOIN会有更多的选择，因为多了t1.a = t3.c的条件。

- Evaluate const expressions

> 计算常量表达式，例如：t.a = t2.b AND t2.b = 10 会被转换为 t1.a = 10。

- Remove trivial conditions

> 移除永真或永假的条件，例如 1=1 这种会被替换为TRUE。如果 t1.a = t2.b OR 1 = 1，这个条件会被整个移除，并被替换为TRUE。

下面几章阅读代码，看一下上面几个步骤的实现过程。

#### 1. Negation elimination - 否定消除



#### 2. Equality propagations - 等式传播

等式传播会将多个等式条件进行合并，合并的条件是这些等式都是相等的，合并的结果是一个字段的集合，集合中的字段都是相等的。举个例子，对于条件：t1.a = t3.a AND t2.a = t4.a AND t1.a = t2.a，在经过等式传播后这个条件会被转换为eq_set(t1.a, t2.a, t3.a, t4.a)，表示t1.a=t2.a=t3.a=t4.a。

##### 2.1 为什么要进行等式传播？

等式传播的意义是可以让后续的优化过程（例如join优化）使用更多的条件信息。例如前面的条件（t1.a = t3.a AND t2.a = t4.a AND t1.a = t2.a）在没有经过等式传播之前只能得到3组等式关系，即：(t1.a=t3.a)，(t2.a=t4.a)，(t1.a=t2.a)，但是经过等式传播之后，我们可以额外得到1组等式关系，即：(t1.a=t4.a)。这个多出来的等式关系可能会给后续优化带来更多的选择。

##### 2.2 MySQL中等式传播的实现



#### 3. Evaluate const expressions - 常量表达式计算



#### 4. Remove trivial conditions - 恒等条件消除



























































