# MySQL中关于JOIN的性能考虑
在很多公司里面，DBA会禁止使用JOIN语句，而是将联表的操作放到后端进行，今天就这个问题来分析一下为什么需要这么做；

# JOIN的执行原理
JOIN的执行原理我们需要分两种大情况讨论：可以使用被驱动表上的索引与不能使用被驱动表上的索引，我们假设现在有两张表A与B，分情况讨论A JOIN B的执行原理

## 当可以使用被驱动表上的索引时
这个时候，这种算法被称为“Index Nested-Loop Join”，简称NLJ算法；
其流程为：

1. 从A中取出每一个满足条件的行
2. 再从每一行取出JOIN时使用的字段
3. 再去搜索B中的对应的相等的字段所在的行，并与A的那一行组合为一行返回
4. 重复1~3直到A表的末尾

这种时候，可以使用B表中的某个字段的索引；也就是说我们的扫描行数其实是**A的全表 + B中满足条件的行数**

我们再分析把这个过程拆分为单表执行，即先查出A中所有满足条件的行，然后再循环A查出B中对应的行数，SQL语句中没有循环，也就是说我们首先 SELECT * FROM A 只需要执行一条语句；
然后我们需要循环执行从B中查到的A中对应的字段相同的行，也就是说我们扫描的行数是一致的，但是我们的交互会进行 1 + A的COUNT数 次，所以效率反而没有JOIN高；

## 当不可以使用被驱动表上的索引时
这个时候，会有两种算法，分别被称为“Simple Nested-Loop Join”和“ Block Nested-Loop Join”（BNL）；

### Simple Nested-Loop Join
很简单，其实就是A表走全表，B表也走全表，这样扫描行数就相当于把A的行数乘以B的行数；

### Block Nested-Loop Join
但是我们可以很明显看出来，这种的效率太过于低下了，所以MySQL也没有使用该算法，而是使用BNL算法

我们先把A表中的符合条件的数据全部取出来放到join_buffer中，然后把表B中的每一行取出来与join_buffer中的数据做对比然后满足条件的作为结果集一部分返回；

但是join_buffer中的数据是无序的，所以A和B也是做全表，也就是说，BNL算法其实扫描行数跟上一个算法并无区别；
但是BNL算法是放到内存中进行计算的，所以效率会高些；

#### Block Nested-Loop Join的Block是什么意思
我们刚刚学到，BNL算法会把驱动表中的数据放到join_buffer中，当join_buffer放不下数据的时候，MySQL会进行怎么样的处理呢？
MySQL会先读取部分的数据，放入到join_buffer，然后与被驱动表进行比较，取出结果集，然后重复这个过程，直到驱动表数据读取完；
