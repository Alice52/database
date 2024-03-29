## Engine- InnonDB

1. InnoDB 是第一个完整支持 ACID 事务的 MySQL 存储引擎: OLTP
2. [feature](../01.basic/01.introduce.md)

   - 行锁设计
   - 外键
   - 支持 MVCC
   - 提供一致性非锁定读
   - 非聚簇索引
   - 崩溃恢复: redolog

3. 内容

   ![avatar](/static/image/mysql/mysql-innodb-architecture-5.7.png)
   ![avatar](/static/image/mysql/mysql-innodb-architecture-8.png)

   - 后台线程: 7 个
   - buffer pool: ahi | insert/change buffer | 数据/缓存页(数组字典信息) | 链表(LRU) | 其他信息(锁)
   - redolog
   - undolog
   - doublewrite buffer:
     1. innodb 默认一 16k/page 读取写入, 磁盘是 4k/page 写入读取, 防止磁盘写入一半而断电无法恢复数据的情况发生, 而引入了 double writer buffer 机制
     2. doublewrite buffer 是一段连续空间, 大小 2M(128page), 数据写入的时候先写到 doublewrite 空间, 然后再写入到磁盘, 如果发生写入了一个 page 一半的时候断电, 恢复后会自动从 doublewrite 中恢复
   - additional memory pool
     ![avatar](/static/image/mysql/mysql-innodb-memory.png)
     ![avatar](/static/image/mysql/mysql-engine-layout.png)

4. innodb flow

   - 把数据库文件按页[每页 16K]读取到缓冲池, 然后按照 LRU 的算法来保存在缓冲池中的缓冲数据
   - 如果数据库需要更改, 顺序也必须是:
     1. 先修改缓冲池中的页`[修改之后的页就是脏页]`, 再按照一定的频率把缓冲池的脏页刷新到文件
     2. 写 redolog 的 prepare 状态
     3. 写 binlog
     4. 调用存储引擎层, 提交事务, 将 redolog 修改为 commit 状态
   - 所以缓冲池是占内存最大的一部分: 作用是存放各种数据的缓存{**索引页/数据页**/~~undo 页~~/插入缓冲/自适应哈希索引/InnoDB 存储的锁信息和数字字典信息}

### 后台线程

1. 主要作用是负责刷新内存池的中的数据, 保证缓冲池中的内存缓存的是最近的数据
2. InnoDB 是个单线程的存储引擎
3. 4 个 IO Thread + 1 master thread + + 1 lock 监控线程 + 1 错误监控线程

   - io 线程: innodb_file_io_threads
   - io 线程: insert buffer thread, log thread, read thread, write thread

4. 每秒任务

   - redolog buffer --> fs page cache --> disk: 可能事务还没有提交也照做不误
   - 合并插入缓冲: 不是每秒都发生, InnoDB 存储引擎会判断当前的一秒内发生的 IO 次数是否小于 5 次{也就是 IO 压力很小}, 则可以执行插入缓冲的操作
   - 刷新 100 个 InnoDB 的缓冲池中的脏页到磁盘: 也不是每秒都发生, InnoDB 存储引擎通过判断当前缓冲池中的脏页的比例, 是否超过了配置文件中的 innodb_max_dirty_pages_pct 这个参数, 如果超过了才会做磁盘同步操作
   - 如果当前没有活动就切换到 background loop;

5. 每 10 秒任务

   - 刷新 100 个脏页到磁盘: InnoDB 存储引擎会先判断过去 10 秒之内磁盘的 IO 操作是否小于 200 次{也就是是否有足够的磁盘 IO 操作能力}, 如果有的话就把 100 个脏页刷新到磁盘
   - 合并至多 5 个插入缓冲: 刷新完脏页之后就会插入合并缓冲
   - 把日志缓冲刷新到磁盘
   - 删除无用的 undo 页: InnoDB 进一步执行 full purge 操作, InnoDB 会判断当前事务系统已经被删除的行是否可以删除, 如果是就立即删除, 一次最多删除 20 个
   - 刷新 100 个或者 10 个脏页到磁盘: 接着 InnoDB 存储引擎会判断缓冲池中脏页的比例, 如果超过 70%, 就刷新 100 个脏页到磁盘, 如果小于 70%, 就刷新 10%的脏页到磁盘
   - 产生一个检查点: InnoDB 检查点也叫模糊检查点, InnoDB 存储引擎在 checkpoint 时不会把所有的缓冲池中的脏页都写入磁盘[这样会影响性能], 而是把 Oldest LSN[最老日志序列号]的页写入磁盘
   - 如果当前没有活动就切换到 background loop;

6. background loop

   - 删除无用的 undo 页
   - 合并 20 个插入缓冲
   - 跳回到主循环
   - 不断刷新 100 个页直到符合条件

7. _自适应哈希索引_
   - 哈希索引的查找复杂度仅为 O(1): 所以速度非常快, 常常用于连接操作
   - 自适应的意思是: InnoDB 存储引擎会监控对表上索引的查找, 如果观察到建立索引表可以带来速度的提升就建立哈希索引

---

## reference

1. https://blog.csdn.net/qq_34408516/article/details/123296196
2. https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html
