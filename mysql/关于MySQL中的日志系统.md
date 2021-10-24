# 关于MySQL中的日志系统
MySQL中的核心日志为redolog和binlog；
其中binlog是MySQL的server层自带的日志系统，主要是记录每一个事务操作时候的原始逻辑；
redolog是Innodb引擎所拥有的日志系统，主要是记录在某个数据页做了什么修改；
redolog和binlog共同实现了MySQL的数据可靠性；

# redolog
redolog是Innodb所拥有的日志系统，其目的是为了使得MySQL拥有crash-safe的功能；
MySQL遵循WAL（Writing-Ahead Logging），相当于先用日志记录，然后再持久化到硬盘；
其redolog做的事其实很简单，简单来说，就是我们的IO操作非常的损耗性能，如果我们每次更新操作，都持久化到磁盘，那么我们必定性能损耗极大；
而有redolog的更新操作，只要我们将更新操作写到redolog中，我们的更新操作其实就已经完成了，Innodb会在“合适的时候”自动持久化到硬盘中；
如果redolog已经满了，那么MySQL会先进行持久化到硬盘的操作；

redolog其实是一个环形的，如果满了则会持久化部分到磁盘中,擦除之前要持久化到磁盘之中；

redolog这么做便实现了即时MySQL异常重启，但是提交的记录也不会丢失的问题，这个能力称为crash-safe；

# binlog
binlog是MySQL的server层自带的日志系统，其主要记录了每一个事务的原始逻辑；
binlog没有crash-safe的能力，所以引入redolog来获得crash-safe的能力；

# 关于两阶段提交
两阶段提交指的是写日志的顺序是按照redolog-prepare，binlog，redolog-commit，三阶不同的阶段进行提交的，而为什么要这么设计呢？
我们都知道，其实redolog是为了拥有crash-safe的能力，也就是说，我们必须要写入到redolog中的事务，才是有效的事务；
binglog是记录的语句的原始执行逻辑，在恢复的时候可以直接执行；
两阶段提交其实是相当于把写这两个事务作为一个原子过程；

我们利用反证法来检验一下两阶段提交的必要性：
## 先写redolog再写binlog
假如我们写入redolog之后程序崩溃，binlog中并没有写入；
也就是说当我们需要恢复数据的时候，我们硬盘里面的数据已经被改变，但是binlog中并没有该条原始逻辑，也就是说会有数据不一致的情况发生；

例：
```sql
UPDATE `user` SET age = age+1 WHERE `user`.id = 1;
```
假设id为1的用户的age=18；
我们正常写入redolog日志，也就是说，我们已经相当于把数据持久化到硬盘中了，此时id为1的用户的age=19；
但是binlog并没有记录该条语句的原始逻辑，也就是说等到恢复数据的时候，id为1的用户的age会变为18；
这样子就会恢复错误的数据；

## 先写binlog再写redolog
加入我们写入binlog之后程序崩溃，并没有写入redolog；
那么也就是说，我们已经保存了原始逻辑，却并没有相应的真正修改到磁盘中；
还是举上面那个例子

例：
```sql
UPDATE `user` SET age = age+1 WHERE `user`.id = 1;
```
假设id为1的用户的age=18；
我们正常写入binlog，记录了这一条语句的原始逻辑，但是磁盘中的数据还是age=18；
这时候假如我们需要恢复数据，则会将age恢复成为17，逻辑又出现了错误；

也就是说，我们必须使用两阶段提交来保持逻辑上的状态一致；
