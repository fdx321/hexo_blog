title: MySQL事务学习总结
date: 2016-09-09 17:08:41
tags:
- MySQL
- 事务
---
关于事务，常看到的概念就是ACID，从单机发展到分布式后，又出现了CAP原理和BASE思想。这里将我最近学习的单机事务做个总结，方便温故知新，后面所有的内容都是基于MySQL/InnoDB的。

|隔离级别|脏读|不可重复读|幻象读|第一类更新丢失|第二类更新丢失|
|----|----|---|----|----|
|READ UNCOMMITED|会|会|会|不会|会|
|READ COMMITED|不会|会|会|不会|会|
|REPEATABLE READ|不会|不会|会|不会|不会|
|SERIALIZABLE|不会|不会|不会|不会|不会|

ANSI/ISO SQL 92标准定义上面4种隔离级别，以及每种隔离级别要达到什么标准。注意，这里是SQL 92的标准，实际数据库的实现和表格中列的是有出入的。例如，对于REAPEATABLE READ隔离级别，Wikipedia上是这么描述的：
**Repeatable reads[edit]**
In this isolation level, a lock-based concurrency control DBMS implementation keeps read and write locks (acquired on selected data) until the end of the transaction. **However, range-locks are not managed, so phantom reads can occur.**
但是对于MySQL/InnoDB， 在 REPEATABLE READ隔离级别下，是不会出现幻象读的，它是通过一种GAP间隙锁来实现的，后面再详细总结。<!--more-->

可以使用*select @@tx_isolation;*来查看当前的隔离级别，
```sql
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)
```
可以使用下面的命令来设置当前会话的隔离级别，全局的可以用*set global trans....*
```sql
mysql> set session transaction isolation level read uncommitted;
Query OK, 0 rows affected (0.00 sec)
```
接下来通过实例演示下各种情况，后面所有的实例都是通过下面这张表来演示的
```sql
mysql> describe test;
+-------+---------+------+-----+---------+-------+
| Field | Type    | Null | Key | Default | Extra |
+-------+---------+------+-----+---------+-------+
| id    | int(11) | NO   | PRI | NULL    |       |
| money | int(11) | YES  | MUL | NULL    |       |
+-------+---------+------+-----+---------+-------+
2 rows in set (0.00 sec)
```
#### 脏读
A事务读取B事务尚未提交的更改数据，并在这个数据的基础上操作。如果恰巧B事务回滚，那么A事务读到的数据根本是不被承认的。

|时间|转账事务A|取款事务B|
|--|--|--|
|T1||开始事务|
|T2|开始事务||
|T3||查询账户余额为1000元|
|T4||取出500元把余额改为500元|
|T5|查询账户余额为500元（脏读）||
|T6||撤销事务余额恢复为1000元|
|T7|汇入200元把余额改为700元||
|T8|提交事务| ||

下图是实际操作的图，左右两边和上表格的对应，从上到下表示操作时序。
![bbb](/images/MySQL事务学习总结_1.png)
**只有在READ UNCOMMITED隔离级别下，才会出现这种情况。正如"读未提交"这个名字锁描述的，正是因为事务A能够读到事务B未提交的数据才会造成脏读的情况。**
#### 不可重复读
|时间|转账事务A|取款事务B|
|--|--|--|
|T1||开始事务|
|T2|开始事务||
|T3||查询账户余额为1000元|
|T4|查询账户余额为1000元||
|T5||取款500元，余额变为500元|
|T6||提交事务|
|T7|查询账户余额为500元|||
下图是实际操作的图，左右两边和上表格的对应，从上到下表示操作时序。
![bbb](/images/MySQL事务学习总结_2.png)
**在READ UNCOMMITED和READ COMMITED隔离级别下，都会出现这种情况。T7和T4的操作在同一个事务里，但是两次读到的数据是不一样的，这就是下个隔离级别"可重复读"要解决的问题。**

#### 幻读
关于幻读，网上很多描述都是错误的。幻象读和不可重复读最大的区别在于，前者是指读到了其它事务提交的**新增**数据，而后者是指读到了其他事务提交的**更改**数据（更改或删除）。为了解决不可重复读，只需要通过行级锁来实现就可以了，但是为了解决幻读，则不能仅仅锁住一条数据，因为这样的锁不能阻止别的事务新增记录，MySQL用了间隙锁来解决这个问题，而不是表级锁。说到这里，可能有人会用下面这样的例子来打我脸了：
![bbb](/images/MySQL事务学习总结_3.png)
**你看你看，在REPEATABLE隔离级别下，左边的事务还不是读到了右边事务新增的数据了吗，不是说MySQL在这种隔离级别下不会出现幻读吗。** 这并不是幻读，InnoDB实现的是MVCC(什么是MVCC呢，不太懂，不敢瞎BB)。在MVCC中，读分为快照读和当前读，上图中的
```sql
select * from test where money = 500;
```
就是快照读。那么哪些操作是属于当前读呢？
```sql
select * from table where ? for update;
insert into table values (…);
update table set ? where ?;
delete from table where ?;
```
可能会觉奇怪，为什么insert update delete也属于当前读，因为针对这些操作，都是InnoDB先把数据筛选出来，加锁，把数据交给MySQL Server处理，Server处理好后交给InnoDB更新，然后InnoDB释放锁，这里面就有读的操作。
这里说的幻读指的是当前读。这是登博关于幻读的解释，***所谓幻读，就是同一个事务，连续做两次当前读 (例如：select * from t1 where id = 10 for update;)，那么这两次当前读返回的是完全相同的记录 (记录数量一致，记录本身也一致)，第二次的当前读，不会比第一次返回更多的记录 (幻象)。**接下来实例操作一下：
![bbb](/images/MySQL事务学习总结_4.png)
可以看到事务隔离级别设置的是可重复读，T4操作是一个当前读，T6,T7,T8的insert操作失败了，T9的insert操作成功了。接下来分析一下为什么会是这样。要了解原因，首先得了解一下索引的结构。

关系型数据库存储数据的方式通常有两种，一种是将数据无序的放在堆表中，然后有索引指向它，叫做非聚簇索引，还有一种是聚簇索引，索引的key是主键，表的数据也存储在该索引的叶子节点上。聚集索引的优点就是按主键索引查找数据很快，因为数据就在B+树的叶子上，而且是有序的，不需要再根据指针去找。但是对于非主键的索引，则需要先关联到聚簇索引上，然后再去找，多了一次索引的查找。擦。。。跑题了，反正大概就这样，InnoDB的索引结构就如图所示，该图引用自《高性能MySQL》。
![bbb](/images/MySQL事务学习总结_5.png)
OK，那么针对我们test表中的数据，我来画一下它大概的索引结构。test表中id是主键，money是非唯一索引。
![bbb](/images/MySQL事务学习总结_6.png)
这里没有画出完整的B+数，只画了叶子节点。如图所示，左边的是聚簇索引的，右边的是一般的索引。在可重复读隔离级别下，针对SQL
```sql
select * from test where money=500 for update;
```
数据库首先会通过右边的索引找到两条记录，然后找到他们对应的主键值，然后再到左边的索引里找到具体的数据。其实左边的索引下面是可以有很多列的，这里演示的test表只有两列。找到记录后，对于两个索引上的数据，都会加X锁。但是这样并不能阻止别的事务insert数据。所以InnoDB在3个红色箭头的地方加了3把锁(间隙锁)，鉴于B+树叶子节点从左到右是有顺序的，这样就能够避免在money=500的数据周边插入新的数据，造成幻读。这也就是为什么T6,T7,T8的insert操作失败了，T9的insert操作成功了的原因。

关于MySQL锁，强烈推荐登博（阿里高级技术专家）的文章[MySQL 加锁处理分析](http://hedengcheng.com/?p=771)

#### 第一类更新丢失
事务A回滚时把事务B提交的数据覆盖了

|时间|取款事务A|转账事务B|
|--|--|--|
|T1|开始事务||
|T2||开始事务|
|T3|查询账户余额为1000元| |
|T4||查询账户余额为1000元|
|T5||汇入100元把余额改为1100元|
|T6||提交事务|
|T7|取出100元把余额改为900元||
|T8|撤销事务||
|T9|余额恢复为1000元（丢失更新）|||

#### 第二类更新丢失
|时间|取款事务A|转账事务B|
|--|--|--|
|T1|开始事务||
|T2||开始事务|
|T3|查询账户余额为1000元| |
|T4||查询账户余额为1000元|
|T5||取出100元把余额改为900元|
|T6||提交事务|
|T7|汇入100||
|T8|提交事务||
|T9|余额变为1100（丢失更新）|||

关于最后两类更新就不实例操作了，太晚了，睡觉。

最后，谢谢何登成的博客，
#### Reference
* [数据库事务](http://www.cnblogs.com/zhangpengme/archive/2011/11/24/2261975.html)
* [Isolation (database systems)](https://en.wikipedia.org/wiki/Isolation_(database_systems))
* [ MySQL数据库事务隔离级别(Transaction Isolation Level)](http://blog.csdn.net/stark_summer/article/details/48824813)
* [高性能MySQL](https://book.douban.com/subject/23008813/)
* [MySQL 加锁处理分析](http://hedengcheng.com/?p=771)
<style>
img[alt="aaa"] {
  width:420px;
  display: block;
}
img[alt="bbb"] {
  width:620px;
  display: block;
}
</style>
