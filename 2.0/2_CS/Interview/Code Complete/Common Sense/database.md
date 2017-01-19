# 数据库

## 事务处理

事务的概念来自于两个独立的需求：并发数据库访问，系统错误恢复。

一个事务是可以被看作一个单元的一系列SQL语句的集合。

### 事务的特性（ACID）

+ A, atomacity 原子性 事务必须是原子工作单元；对于其数据修改，要么全都执行，要么全都不执行。通常，与某个事务关联的操作具有共同的目标，并且是相互依赖的。如果系统只执行这些操作的一个子集，则可能会破坏事务的总体目标。原子性消除了系统处理操作子集的可能性。
+ C, consistency 一致性。事务将数据库从一种一致状态转变为下一种一致状态。也就是说，事务在完成时，必须使所有的数据都保持一致状态（各种 constraint 不被破坏）。
+ I, isolation 隔离性 由并发事务所作的修改必须与任何其它并发事务所作的修改隔离。事务查看数据时数据所处的状态，要么是另一并发事务修改它之前的状态，要么是另一事务修改它之后的状态，事务不会查看中间状态的数据。换句话说，一个事务的影响在该事务提交前对其他事务都不可见。
+ D, durability 持久性。事务完成之后，它对于系统的影响是永久性的。该修改即使出现致命的系统故障也将一直保持。

### 事务的隔离级别

如果不对数据库进行并发控制，可能会产生异常情况：

1. 脏读(Dirty Read)
    + 当一个事务读取另一个事务尚未提交的修改时，产生脏读。
    + 同一事务内不是脏读。 一个事务开始读取了某行数据，但是另外一个事务已经更新了此数据但没有能够及时提交。这是相当危险的，因为很可能所有的操作都被回滚，也就是说读取出的数据其实是错误的。
2. 非重复读(Nonrepeatable Read) 一个事务对同一行数据重复读取两次，但是却得到了不同的结果。同一查询在同一事务中多次进行，由于其他提交事务所做的修改或删除，每次返回不同的结果集，此时发生非重复读。
3. 幻像读(Phantom Reads) 事务在操作过程中进行两次查询，第二次查询的结果包含了第一次查询中未出现的数据（这里并不要求两次查询的SQL语句相同）。这是因为在两次查询过程中有另外一个事务插入数据造成的。
    + 当对某行执行插入或删除操作，而该行属于某个事务正在读取的行的范围时，会发生幻像读问题。
4. 丢失修改(Lost Update)
    + 第一类：当两个事务更新相同的数据源，如果第一个事务被提交，第二个却被撤销，那么连同第一个事务做的更新也被撤销。
    + 第二类：有两个并发事务同时读取同一行数据，然后其中一个对它进行修改提交，而另一个也进行了修改提交。这就会造成第一次写操作失效。

为了兼顾并发效率和异常控制，在标准SQL规范中，定义了4个事务隔离级别，（ Oracle 和 SQL Server 对标准隔离级别有不同的实现 ）

1. 未提交读(Read Uncommitted)
    + 直译就是"读未提交"，意思就是即使一个更新语句没有提交，但是别的事务可以读到这个改变。
    + Read Uncommitted允许脏读。
2. 已提交读(Read Committed)
    + 直译就是"读提交"，意思就是语句提交以后，即执行了 Commit 以后别的事务就能读到这个改变，只能读取到已经提交的数据。Oracle等多数数据库默认都是该级别。
    + Read Commited 不允许脏读，但会出现非重复读。
3. 可重复读(Repeatable Read)：
    + 直译就是"可以重复读"，这是说在同一个事务里面先后执行同一个查询语句的时候，得到的结果是一样的。
    + Repeatable Read 不允许脏读，不允许非重复读，但是会出现幻象读。
4. 串行读(Serializable)
    + 直译就是"序列化"，意思是说这个事务执行的时候不允许别的事务并发执行。完全串行化的读，每次读都需要获得表级共享锁，读写相互都会阻塞。
    + Serializable 不允许不一致现象的出现。

### 事务隔离的实现——锁

1. 共享锁(S锁)
    + 用于只读操作(SELECT)，锁定共享的资源。共享锁不会阻止其他用户读，但是阻止其他的用户写和修改。
2. 更新锁(U锁)
    + 用于可更新的资源中。防止当多个会话在读取、锁定以及随后可能进行的资源更新时发生常见形式的死锁。
3. 独占锁(X锁，也叫排他锁)
    + 一次只能有一个独占锁用在一个资源上，并且阻止其他所有的锁包括共享缩。写是独占锁，可以有效的防止“脏读”。

Read Uncommited 如果一个事务已经开始写数据，则另外一个数据则不允许同时进行写操作，但允许其他事务读此行数据。该隔离级别可以通过“排他写锁”实现。

Read Committed 读取数据的事务允许其他事务继续访问该行数据，但是未提交的写事务将会禁止其他事务访问该行。可以通过“瞬间共享读锁”和“排他写锁”实现。

Repeatable Read 读取数据的事务将会禁止写事务（但允许读事务），写事务则禁止任何其他事务。可以通过“共享读锁”和“排他写锁”实现。

Serializable 读加共享锁，写加排他锁，读写互斥。

## 索引


数据库创建索引能够大大提高系统的性能。

1. 通过创建唯一性的索引，可以保证数据库表中每一行数据的唯一性。
2. 可以大大加快数据的检索速度，这也使创建索引的最主要的原因。
3. 可以加速表和表之间的连接，特别是在实现数据的参考完整性方面特别有意义。
4. 在使用分组和排序子句进行数据检索时，同样可以显著的减少查询中查询中分组和排序的时间。
5. 通过使用索引，可以在查询的过程中，使用优化隐藏器，提高系统的性能。

增加索引也有许多不利的方面。

1. 创建索引和维护索引需要消耗时间，这种时间随着数量的增加而增加。
2. 索引需要占物理空间，除了数据表占据数据空间之外，每一个索引还要占一定的物理空间，如果要建立聚簇索引，那么需要额空间就会更大。
3. 当对表中的数据进行增加，删除和修改的时候，索引也要动态的维护，这样就降低了数据的维护速度。

应该对如下的列建立索引

1. 在作为主键的列上，强制该列的唯一性和组织表中数据的排列结构。
2. 在经常用在连接的列上，这些列主要是一些外键，可以加快连接的速度。
3. 在经常需要根据范围进行搜索的列上创建索引，因为索引已经排序，其指定的范围是连续的。
4. 在经常需要排序的列上创建索引，因为索引已经排序，这样查询可以利用索引的排序，加快排序查询时间。
5. 在经常使用在where子句中的列上面创建索引，加快条件的判断速度。

有些列不应该创建索引

1. 在查询中很少使用或者作为参考的列不应该创建索引。
2. 对于那些只有很少数据值的列也不应该增加索引（比如性别，结果集的数据行占了表中数据行的很大比例，即需要在表中搜索的数据行的比例很大。增加索引，并不能明显加快检索速度）。
3. 对于那些定义为text，image和bit数据类型的列不应该增加索引。这是因为，这些列的数据量要么相当大，要么取值很少。
4. 当修改性能远远大于检索性能时，不应该创建索引，因为修改性能和检索性能是矛盾的。

创建索引的方法：直接创建和间接创建（在表中定义主键约束或者唯一性约束时，同时也创建了索引）。

索引的特征：

唯一性索引和复合索引。唯一性索引保证在索引列中的全部数据是唯一的，不会包含冗余数据。复合索引就是一个索引创建在两个列或者多个列上。可以减少一在一个表中所创建的索引数量。

