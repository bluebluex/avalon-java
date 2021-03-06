### Mysql

### 索引的基础知识

- `Mysql` 的基本存储结构是页：`16KB`

  - 各个**数据页**可以组成一个**双向链表**：**相邻数据与编号可能并不是连续的，也就是说我们使用的这些页再存储空间里可能并不是连续(挨着)的**；它们只是通过维护着上一个页和下一个页的编号而建立了链表关系；
  - **下一个数据页中用户记录的主键值必须大于上一个页中用户记录的主键值**
  - 每个数据页中的**记录**又可以组成一个**单项链表**：每个**数据页都会为存储在它里边的记录**生成一个**页目录**，在通过**主键**查找某条记录的时候，可以在**页目录中使用二分法**快速定位到对应的槽，然后(依旧使用的是二分法，因为数据页中的记录会根据主键从小到大排好序的单向链表)再遍历该槽对应**分组中的记录**即可快速找到指定的记录

  - 以其他列(非主键)作为搜索条件：只能从最小记录开始开始依次遍历单链表中的每条记录。

`select * from user where username = 'avalon'` 无索引优化的 `sql` 语句：

- 页与页之间是双向链表，页内是单链表；需要沿着双向链表，从第一页开始，一次遍历每一页，最终找到对应的数据

### 聚集和非聚集索引

区别：

- 聚集索引再叶子节点存储的是表中的**数据** **/** 非聚集索引在叶子索引上存储的是**[索引列+主键]**
- 使用非聚集索引查询出数据时，拿到叶子上的**主键**再去查想要查找的**数据(回表)**

联合索引(覆盖索引)&唯一索引：联合索引不一定回表(如果查询的信息只是覆盖索引的信息(字段)而已的话)

### 索引总结

- 最左前缀匹配原则：`MySQL` 会一直从左向右匹配制导遇到范围查询(>, < , BETWEEN, LIKE)
- 尽量选择区分度高的列作为索引，区分度的公式 `COUNT(DISTINCT col)/COUNT(*)`
- 索引列不能参与计算：B+树中存储的都是数据表中的字段值，如果加了函数需要对存储的元素都应用函数才能比较，代价太大！

### 数据库锁

数据库隐式加锁：

- 对于`update、delete、insert` 语句，`InnoDB` 会自动给设计数据集加排他锁

- `MySAM` 执行 `select` 语句前，会自动给涉及的所有表加读锁，执行写操作(`update、delete、insert`) 会自动给涉及表加写锁

  ### 行锁：InnoDB

  `InnoDB` 行锁类型：

  - 共享锁(读锁)：允许一个事务去读一个行，阻止**其他事务**获得相同数据集**排他锁(写锁)** 支队其他客户端
  - 排他锁(写锁)：允许获得排他锁的事务(主体)更新数据；阻止其他事务获取相同数据集的共享(读)锁和排他(写) 锁；

  ### `MVCC` 和事务隔离级别

  MVCC(多版本并发控制)：行级锁的升级版

  - 事务的**隔离级别**是通过**锁的机制**来实现的
  - MVCC实现的读写不阻塞：多版本并发控制 -> 通过一定机制生成一个数据请求时间点的 **一致性数据快照(Snapshot)**，并用这个快照来提供一定级别(**语句级-[Read-Commit]或事物级[Repeatable-Read]**)的**一致性读取**，从用户的角度来看，好像数据库可以提供同一数据的多个版本

  事务隔离级别：

  - Read-Uncommitted：**一个事务读取到另一个事务未提交的数据**；本质就是因为**写操作修改完数据就立即释放锁**，导致读的数据就变成了无用或者错误的数据；
  - Read-Committed：一个事务读取到另一个事务已经提交的数据，也就是说一个事务可以看到其他事务所做的修改；能避免脏读是因为：**写操作之后释放锁的位置调整到事务提交之后**
  - Repeatable-Read：避免不可重复度是因为：**快照的时候是事务级别的**；每次读取的都是当前事务的版本，即使数据行被其他写操作修改了，也只会读取当前事务版本的数据；

### **事务相关**

InnoDB 里面每个事物有一个唯一的事务 ID (transaction id)，它是在事务开始的时候想 InnoDB 的事务系统申请，是按申请顺序严格递增的；

每行数据也都是有多个版本的，每次事务跟新数据的时候，队徽生成一个新的数据版本，并且把 transaction id 赋值给这个数据版本的事务 ID，记为 row trx_id；同时旧的数据版本要保留，并且在新的数据版本中，能够通过信息可以直接拿到它，作为 undo log (回滚)；

一个事务只需要在启动的时候，找到所有已经提交的事务 ID 的最大值，记为 up_limit_id;

**对于可重复读，查询只承认事务启动前就已经提交完成的数据**

**对于读提交，查询只承认在语句启动前就已经提交完成的数据**

**当前读：总是读取已经提交完成的最新版本的数据**

判断可见性两个规则：一个是up_limit_id ,另一个是“自己修改的”；这里用到第二个规则；

Begin之后的第一个语句算启动事务

```
mysql> set global transaction isolation level repeatable read;
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set, 1 warning (0.00 sec)
#A:
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select k from t_learn_tx where id = 1;
+------+
| k    |
+------+
|    1 |
+------+
1 row in set (0.00 sec)

mysql> select k from t_learn_tx where id = 1;
+------+
| k    |
+------+
|    1 |
+------+
1 row in set (0.00 sec)

mysql> commit;
Query OK, 0 rows affected (0.00 sec)
#2
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select k from t_learn_tx where id = 1;
+------+
| k    |
+------+
|    1 |
+------+
1 row in set (0.00 sec)

mysql> select k from t_learn_tx where id = 1;
+------+
| k    |
+------+
|    1 |
+------+
1 row in set (0.00 sec)
## 数据库：更新数据多事先读后写的，而这个读，只能读当前的值，成为"当前读(current read)"，类似JMM内存模型里面的关键字violet
## 出了update语句外，select语句如果加锁，也是当前读
## 判断可见性两个规则：一个是up_limit_id ,另一个是“自己修改的”；这里用到第二个规则
mysql> update t_learn_tx set k = k+1 where id = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select k from t_learn_tx where id = 1;
+------+
| k    |
+------+
|    3 |
+------+
1 row in set (0.00 sec)

mysql> commit;
Query OK, 0 rows affected (0.00 sec)
#3
 update t_learn_tx set k = k+1 where id = 1;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from t_learn_tx;
+----+------+
| id | k    |
+----+------+
|  1 |    2 |
|  2 |    2 |
+----+------+
2 rows in set (0.00 sec)
```





### 主要配置文件 

查询日志：

- 二进制日志log-bin：主要主从复制
- 错误日志log-error：默认是关，巨鹿严重的警告和错误信息，每次启动和关闭的详细信息等
- 查询日志：sql 慢查询

数据文件：

- frm文件：存放表结构
- myd：存放表数据
- myi：存放表索引

### mysql性能优化

优化分析：

- 性能下降sql慢/执行时间长/等待时间长
  - 查询语句写的烂：
  - 索引失效
  - 关联太多join
  - 服务器调优及各个参数设置(缓冲/线程数)

- 常见通用的join查询
  - sql的执行顺序：FROM ON - JOIN WHERE - GROUP BY - HAVING - SELECT - ORDER BY - LIMIT	

