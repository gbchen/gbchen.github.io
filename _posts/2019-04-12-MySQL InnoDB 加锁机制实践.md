---
layout:     post
title:      MySQL InnoDB 加锁机制
subtitle:   数据库加锁机制
date:       2019-04-12
author:     wxj
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
   - InnoDB
   - MySQL
   - 数据库
   - 锁
---

#                     MySQL InnoDB 加锁机制

## 1 InnoDB 存储引擎 

* InnoDB是MySQL5.5.8版本后默认的存储引擎，支持事务、行锁设计、外键以及非锁定读
* InnoDB实现了MySQL标准的4种隔离级别，默认隔离级别是Repeatable Read，并且使用next-key锁来防止幻读。
* 通过主键聚集表数据（聚集索引clustered）



###  1.1 聚集索引

​        聚集索引：文件是按照索引值顺序排列的，一般是表的主键。

######         辅助索引：索引值顺序和文件顺序不一定一致，并且在辅助索引中不会保存行的物理位置，而是保存的主键。因此当根据辅助索引查找时，是先找到主键，再找到记录，要进行二次查询，所以又称为二级索引。

###  1.2 B+Tree

​           当文件数据量很大的时候，索引可能会非常大，所以为了减少内存的占用，索引也会被存储在磁盘上。

> ​      在 MySQL 中，主要有四种类型的索引，分别为： **B-Tree 索引， Hash 索引， Fulltext 索引和 R-Tree 索引**。 B/B+树是实现索引常用的数据结构。它是一个平衡的多叉查找树，从根节点到每一个叶子节点的距离相等且树的高度可控，因此能够大幅度的减少查找次数。

![b tree](https://user-images.githubusercontent.com/32808358/31586660-7bb80142-b206-11e7-8559-b6536f02ec3e.jpeg)



​        InnoDB 存储引擎实际使用的存储结构是 B+Tree，在 B-Tree 数据结构的基础上做了很小的改造。所有实际需要的数据都存放于 Tree 的 Leaf Node(叶子节点) ，而且到任何一个 Leaf Node 的最短路径的长度都是完全相同的。每一个Leaf Node 上面除了存放索引键的相关信息之外，还存储了指向与该 Leaf Node 相邻的后一个 LeafNode 的指针信息（增加了顺序访问指针）。



## 2 InnoDB加锁分析

### 2.1 snapshot read && current read

* 1 快照读：不加锁，读取的是记录的可见版本 ，有可能是历史版本，数据可能不是最新
* 2 当前读：读取的是记录的最新版本，并且对返回的记录会加上锁，保证其他事务不会再并发修改这条记录。

select * from table —— 快照读 

select * from table for update ——当前读

select * from table lock in share mode ——当前读

update / delete / insert ——当前读

下面看一个例子：

***

session 1 : 第一个事务，查询 id = 2 的数据。注意此处没有commit，事务没有结束，还在进行中

<img width="630" alt="1" src="https://user-images.githubusercontent.com/32808358/31583905-0e8dbaf6-b1d6-11e7-83e4-2ec28c1e73f4.png" align='left'>

***

session 2：执行第二个事务，update id = 2 的记录，并且提交更新

<img width="942" alt="2" src="https://user-images.githubusercontent.com/32808358/31583852-d45c8854-b1d4-11e7-90c0-de7546a1e3a8.png">

***

session 1：回到第一个还在进行中的事务，可以看到select查询的数据并非是最新数据，select for update能够看到当前已经提交的最新数据。

<img width="968" alt="3" src="https://user-images.githubusercontent.com/32808358/31583853-d490f152-b1d4-11e7-860f-5c00c0d8fddc.png">

第一个事务开始时间要早于第二个事务，因此在第一个事务中，快照读看不到第二个事务提交的更新。

***

​        **InnoDB实现并发控制的方式有两种：多版本并发控制（MVCC），应用场景是snapshot read 快照读，和基于锁的并发控制，应用场景是current read 当前读。下述InnoDB加锁机制，均是针对current read当前读来分析的。**

### 2.2 InnoDB 加锁机制（current read）

####2.2.1 Lock Mode : Shared or Exclusive

* 1 shared lock : 共享读锁 (s)

  允许事务读取一行记录，并且阻塞其他事务获取相同数据集的排他锁

  应用场景：select * from table lock in share mode

* 2 exclusive lock ： 排他写锁 (x)

  允许事务更新记录，阻止其他事务取得相同数据集的共享读锁和排他写锁

  应用场景：select for update 、 delete 、 update 、 insert

#### 2.2.2 Record-Level Lock

​        InnoDB通过对索引加锁来实现行级锁，以锁住索引记录。InnoDB的行级锁，包括记录锁、间隙锁（锁住记录之间的间隙）、next-key锁（记录锁和间隙锁的组合）。

​        除了行级锁外，表锁在特定场景下也会被使用。

* **1 index-record lock : 记录锁**

  锁住满足搜索条件的索引记录 ：lock index records the search encounters.

  *engine innodb status* : lock_mode X(S) , locks rec but not gap，表示行级锁是记录锁，没有间隙锁

* **2 gap lock ：间隙锁**

  锁住满足搜索条件的索引记录的间隙：lock on a gap between index records.

   *engine innodb status* : lock_mode X(S) locks gap，表示行级锁是间隙锁

  需要注意2点：

  **（1）间隙锁可以避免insert新记录 ，是innodb在RR隔离级别下解决幻读的方法**

  **（2）间隙锁不会阻塞其他事务获取间隙锁，只会阻塞insert**

  > Gap locks in `InnoDB` are “purely inhibitive”, which means they only stop other transactions from inserting to the gap. They do not prevent different transactions from taking gap locks on the same gap. 

  通过一个例子来验证间隙锁不会阻塞其他事务获取间隙锁。

  ***

  这是transaction表结构：id主键、order_id 二级唯一索引、user_id 二级非唯一索引

  <img width="977" alt="default" src="https://user-images.githubusercontent.com/32808358/31584405-7f64e9e8-b1e0-11e7-9d08-d9af17206d52.png">

  

  ***

  session 1：开启第一个事务

  ​       以type字段为条件执行update，由于type不是索引，所以这个时候，InnoDB会对数据库表transaction 加表锁（表锁会对表中所有记录加锁，同时对所有记录的间隙加间隙锁）。

  <img width="936" alt="default" src="https://user-images.githubusercontent.com/32808358/31584411-b1eb1c02-b1e0-11e7-94ac-6289c438d291.png">

  ***

  session 2：然后开启第二个事务，此时第一个事务仍在进行中

  ​        先执行select id = 2 for update，这个时候可以看到出现锁竞争。这是因为事务二需要等待id = 2的记录锁（排它锁），但id = 2的记录被事务一加上了记录锁（排它锁），因此出现锁等待超时，sql执行失败。

  ​        然后尝试select id = 8 for update（事务一、二均未关闭，仍在进行，事务一开始时间早于事务二），sql可以执行成功，这是因为id = 8的记录在表中不存在，导致事务二要竞争id 在3-10之间的间隙锁，尽管这块间隙在事务一中被锁住了，但不会被阻塞，这就是间隙锁的特点。

  <img width="1208" alt="default" src="https://user-images.githubusercontent.com/32808358/31584428-1ee70258-b1e1-11e7-9b1e-bf7428253ead.png">

  ***

  ​       出现锁竞争时，查看innodb_locks可以看到在排队获取锁的多个事务，下图中展示的是：session 2中 select id = 2 for update 时候，出现的记录锁竞争。

  <img width="900" alt="default" src="https://user-images.githubusercontent.com/32808358/31584421-dc836f6e-b1e0-11e7-8600-124fee3a16fe.png">

  ​       表Innodb_locks：

  （1）只有出现锁竞争时，表中才会有数据，如果有一个或多个事务在同时进行，但并没有出现锁竞争，是看不到数据的

  （2）出现锁竞争时，只能看到被竞争的特定的锁，一个事务中持有的全部的锁，在此处看不全，需要配合其他命令来查看

  ***

  ​       查看innodb_trx可以看到当前在进行中的事务。innodb_locks中的lock_trx_id 是innodb_trx表的trx_id；对running状态的事务，lock_id是事务正在持有的锁id，对lock wait 状态的事务，lock_id是事务正在等待的锁id，对应于innodb_trx表的trx_request_lock_id。

  ​      可以看到，事务二的select id = 2 for update 被阻塞住了。

  <img width="1106" alt="default" src="https://user-images.githubusercontent.com/32808358/31584475-f35fd79e-b1e1-11e7-9911-5d91f682ad3a.png">

* 3 next-key  lock:  前两种锁的结合，锁定一个索引范围及索引记录本身

  > A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.

  应用场景：无索引或非唯一索引，具体实例在2.4节中分析

   *engine innodb status* : transaction lock_mode X ，表示行级锁是next-key锁，包括记录锁和间隙锁

  

### 2.3 Locks Set by Different SQL Statement 

* <img width="1016" alt="state1" src="https://user-images.githubusercontent.com/32808358/31585945-c8153186-b1fc-11e7-9b71-2fd7f57c8ba1.png">

  **select .. lock in share mode : 加共享读锁 S**

  **当对唯一索引进行当前读时，会对查询到的唯一记录加记录锁**

  **其他情况下加的都是next-key lock：记录锁+间隙锁**

  ***

* <img width="1014" alt="state1-2" src="https://user-images.githubusercontent.com/32808358/31585944-c7d99a36-b1fc-11e7-8c46-9464c856343d.png">

* <img width="1009" alt="state2" src="https://user-images.githubusercontent.com/32808358/31585946-c84ee106-b1fc-11e7-9761-4bf765e69e7a.png">

* <img width="999" alt="state3" src="https://user-images.githubusercontent.com/32808358/31585947-c888fe7c-b1fc-11e7-8eb5-3b6ef7cb2d8d.png">

  **select for update 、 update 、delete ：加排它锁X**

  **当对唯一索引进行当前读时，会对查询到的唯一记录加记录锁**

  **其他情况下加的都是next-key lock：记录锁+间隙锁**

  ***

* <img width="1031" alt="state4" src="https://user-images.githubusercontent.com/32808358/31585948-c8c39f6e-b1fc-11e7-9448-1dd05c993530.png">

  **Insert 完成后会对记录加记录锁（排它锁），而在insert前，需要先获取insert intention gap lock。**

  **insert intention gap lock 不会阻止其他事务获取相同间隙的间隙锁，只要Insert记录不重复**

  （间隙锁不会阻塞其他事务获取间隙锁，但是会阻塞insert ，是在于间隙锁会阻塞的是insert intention gap lock 这种类型，除此之外的间隙锁都不被阻塞）

### 2.4 Different Search Condition with Index

*  2.4.1**主键索引 primary key  ：加记录锁，当主键索引值不存在时，会加间隙锁**

  ***

  session 1：开启第一个事务，根据id = 8进行update ,这条记录不存在，因此对 3<id<10的间隙，加间隙锁

  <img width="650" alt="primary key0" src="https://user-images.githubusercontent.com/32808358/31586034-1b390012-b1fe-11e7-86f8-7c1146e15318.png">

  ------

  session 2：开启第二个事务

  ​        插入id = 5的记录，发现出现锁等待。这是因为，insert前需要获取 3-10之间的间隙锁，事务一中已经对此间隙加锁，并且间隙锁阻塞的即为insert（防止幻读）所以Insert id 5 执行失败。

  <img width="1244" alt="primary key1" src="https://user-images.githubusercontent.com/32808358/31586035-1b9eddc4-b1fe-11e7-80ac-a50c0c8e1019.png">

  

  ​        事务二继续，insert id = 0 的记录，sql执行成功，因为此时Insert 竞争的是负无穷到1之间的间隙锁。

  <img width="1254" alt="primary key3" src="https://user-images.githubusercontent.com/32808358/31586038-1c0d0de4-b1fe-11e7-814d-a6c3267aad3a.png">

  ------

   查看lock wait：这是事务二中第一条sql ：insert id = 5 时，出现的锁竞争<img width="1013" alt="primary key" src="https://user-images.githubusercontent.com/32808358/31586033-1aff68f2-b1fe-11e7-9e35-c240b4dad06e.png">

  ------

  查看engine status: 可以看到 lock_mode X locks gap before rec insert intention waiting，表示排它锁X模式的间隙锁，阻塞了插入意向锁（也是间隙锁）

  <img width="1677" alt="primary key2" src="https://user-images.githubusercontent.com/32808358/31586036-1bd51344-b1fe-11e7-8b10-b6719da7e53c.png">

  

* **2.4.2 二级唯一索引：unique key  索引记录加锁&&主键索引加锁**

  > *If a secondary index is used in a search and index record locks to be set are exclusive, `InnoDB` also retrieves the corresponding clustered index records and sets locks on them.*

  ***

  session 1 : 开启第一个事务，根据二级索引order_id来做update，其中order_id是唯一索引

  <img width="1076" alt="unique1" src="https://user-images.githubusercontent.com/32808358/31586240-e131c3b0-b200-11e7-8332-4e442c8cf155.png">

  ***

  session 2：开启第二个事务，对同一条记录根据order_id做update

  <img width="1350" alt="unique2" src="https://user-images.githubusercontent.com/32808358/31586241-e1626a9c-b200-11e7-9180-a842c643dc50.png">

  ***

  查看lock wait：lock_index表示索引项order_id的对应记录被锁住

  <img width="1007" alt="unique3" src="https://user-images.githubusercontent.com/32808358/31586242-e19412b8-b200-11e7-948d-c72cbf6ac38c.png">

  ***

  查看engine status: 提示信息 record locks index index_order_id of table `test`  `transaction`

  <img width="1680" alt="unique4" src="https://user-images.githubusercontent.com/32808358/31586244-e1c58744-b200-11e7-96f7-9ed5810022f8.png">

**----------------------------------同时，对应的主键索引记录也被锁住了----------------------------------**

​        事务二中继续操作（事务一、二均在进行，事务一开始时间早于事务二。事务一中根据唯一索引order_id做update，事务二中根据同一索引的同一索引值做update，未执行成功）

​        在事务二中，根据order_id = 222的这条记录中的主键id = 2来做update，发现同样出现锁等待。

<img width="1255" alt="unique5" src="https://user-images.githubusercontent.com/32808358/31586245-e1f57954-b200-11e7-8a40-21be112440cf.png">

------

 查看lock wait：lock_index提示主键索引被锁住了。

<img width="1004" alt="unique6" src="https://user-images.githubusercontent.com/32808358/31586246-e2254792-b200-11e7-893b-07e38de94dcc.png">

------

 查看engine status:提示信息 record locks index PRIMARY of table `test``transacition`，即表的主键索引被锁住（注意对比，上述提过的index_order_id被锁住）

<img width="2000" alt="unique7" src="https://user-images.githubusercontent.com/32808358/31586247-e25502fc-b200-11e7-95a0-b1f3777e4966.png">



* **2.4.3 二级非唯一索引：key**

  session 1 : 开启第一个事务，根据user_id 做update，user_id是索引，但不是唯一索引

  <img width="800" alt="key 1" src="https://user-images.githubusercontent.com/32808358/31586506-8d51fb30-b204-11e7-8a47-0561f5bca539.png">

  ------

  session 2：开启第二个事务，插入id = 5，user_id = 55的记录，出现锁等待超时

  <img width="1204" alt="key 2" src="https://user-images.githubusercontent.com/32808358/31586507-8d803b76-b204-11e7-8d7e-cdcb8fa38ba3.png">

  ------

  查看lock wait：发现index_user_id索引上，被加上了X锁的GAP锁，锁住了user_id在2-110之间的区间，因此阻塞了user_id = 55的插入。

  <img width="995" alt="key 3" src="https://user-images.githubusercontent.com/32808358/31586508-8daf70a8-b204-11e7-9885-fe22a2965e7f.png">

  ------

  查看engine status：提示user_id索引上，间隙锁阻塞了insert。

  <img width="1665" alt="key 4" src="https://user-images.githubusercontent.com/32808358/31586509-8dde0120-b204-11e7-917c-8b0b9b31e485.png">

  **二级非唯一索引上的当前读：**

  **（1）对满足条件的一条或多条索引记录，加记录锁——包括索引上的记录锁，和索引记录对应的主键索引上的记录锁（辅助索引/二级索引作为查询条件时，每条记录上都会加索引的记录锁&&主键的记录锁）**

  **（2）同时会对二级非唯一索引的索引值所在区间，加间隙锁，阻止插入索引值在区间内的记录，而导致其他事务根据此索引做查询时，两次查询查出的记录数量不一致，造成幻读**

*  **2.4.4 无索引 加表锁（所有记录的记录锁，以及gap锁）**

  > *If you have no indexes suitable for your statement and MySQL must scan the entire table to process the statement, every row of the table becomes locked, which in turn blocks all inserts by other users to the table.*

  **表锁会阻断所有并发事务的更新、删除或插入（但是不会影响获取gap锁的update 或 delete）**

  参考例子在2.2.2中

  

###**2.5 加锁总结**

- 主键索引：对主键索引加记录锁
- 唯一索引：对唯一索引以及唯一索引上的主键加记录锁（2个）
- 如果主键索引或唯一索引的索引值不存在，加间隙锁


- 非唯一索引：加 next-key 锁

- 非索引：表锁——等同于所有记录加记录锁，并且记录间隙加间隙锁

   （注意，以上不包括Insert，Insert较特殊）

***

- insert 前：需要获取insert intention gap lock (间隙锁，X mode)

  Insert 成功： 对当前记录加记录锁

  其他事务发生duplicate key insert ，会等待当前记录的next-key lock ( S mode) 

  

### **2.6 Dead Lock 死锁**

场景：insert duplicate key 

session 1 : insert id = 7 —— 先获取insert intention gap lock，成功后进行insert ，持有记录锁

session 2:  insert id = 7  ——等待 id = 7 这条记录及对应gap的 next-key锁，是共享读锁模式

session 3 : insert id = 7  ——等待 id = 7 这条记录及对应gap的 next-key锁，是共享读锁模式

需要竞争id=7记录的锁，所以出现lock wait

<img width="828" alt="deadlock0" src="https://user-images.githubusercontent.com/32808358/31592952-411466d6-b25e-11e7-8e52-e572c8624158.png">



然后对session 1 rollback 。这个时候，session 2 和 session3 都获取next-key（S）锁成功

**发生死锁：**

**session 3 持有 gap的s锁，阻塞session 2 的insert intention gap lock**

**session 2 持有 gap的s锁，阻塞session 3 的insert intention gap lock**

<img width="1657" alt="deadlock1" src="https://user-images.githubusercontent.com/32808358/31592953-4151d016-b25e-11e7-8630-6b5919116eca.png">



<img width="1678" alt="deadlock2" src="https://user-images.githubusercontent.com/32808358/31592954-427e5d06-b25e-11e7-8aa5-a7e8b5682842.png">





### 2.7 思考

问题1 ：在一个事务中，如果进行update做当前读，在已经获取锁后，select 是否也可以读到当前最新数据

Session 1 : select * where id = 2

session 2 : update id = 2 && commit (id = 2 被更新)

回到仍在进行中的事务一中

session 1 :update where type = 999 (表锁)

session 1 :select * where id = 2

***

问题2：主键的gap可以通过主键顺序确认，辅助索引（二级索引）的gap如何确认，开闭区间？



***

问题3：死锁的例子中，为什么session 1 commit 不会导致死锁？

（当有事务提交时，还在进行中的事务会触发一次事务restart，重新读取当前最新数据。这个时候，session 2 和 session 3 已经不再是insert duplicate key的场景了，因此之前持有的S next-key锁会释放掉。）

***



## 3 InnoDB实现并发控制

​       InnoDB实际提供了2种事务隔离技术：MVCC && 基于锁的并发控制。MVCC 用于snapshot read ，基于锁的并发控制用于current read，在使用不同语句的时候，可以动态的选择。

​       优势：读不加锁，读写不冲突。



## 4 命令行

**select @@tx_isolation **

**查看隔离级别**

**set tx_isolation='repeatable-read'** 

------

**SHOW VARIABLES LIKE '%AUTOCOMMIT%'  **

**查看是否自动提交**

**（在设置autocommit = 0 后，执行commit后便开始新的事务，结束当前事务）**

------

**SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX;**

**正在InnoDB引擎中执行的所有事务的信息，包括waiting for a lock和running的事务**

***

**SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;**

**InnoDB事务锁的具体情况，包括事务正在申请加的锁和事务加上的锁（有锁竞争时）**

***

**SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS;**

**被blocked的事务的锁等待的状态**

------

**Show engine innodb status;**

**可用来查看活跃事务及其持有锁的状态 Transaction List  ，还可以进行死锁分析 DeadLock**

***

SELECT
​     a.trx_id,
​     a.trx_state,
​     a.trx_query,
​     b.ID,
​     b.DB,
​     b.COMMAND,
​     c.PROCESSLIST_USER,
​     c.PROCESSLIST_DB,
​     d.SQL_TEXT
FROM
​     information_schema.INNODB_TRX a
LEFT JOIN information_schema.PROCESSLIST b ON a.trx_mysql_thread_id = b.id
AND b.COMMAND = 'Sleep'
LEFT JOIN PERFORMANCE_SCHEMA.threads c ON b.id = c.PROCESSLIST_ID
LEFT JOIN PERFORMANCE_SCHEMA.events_statements_current d ON d.THREAD_ID = c.THREAD_ID;

**查看当前进行中的事务ID、事务的mysql线程状态及执行的SQL Statement**

***

