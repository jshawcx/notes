[toc]

### 一条查询语句如何执行
- mysql基本架构
    - server层
        - 连接器
            ```
            mysql -h$ip -P$port -u$root -p
            show processlist
            ```
            连接mysql建议使用连接池来管理长连接，
        - 缓存（8.0废弃）
        - 分析器
            - 词法分析，识别出关键字、表、列
            - 语法分析
        - 优化器
            - 表里有多个索引，决定用哪一个
            - 多表查询，决定表的先后
        - 执行器
            - 权限检查
            - 根据表的引擎，调用引擎提供的接口
                - 没有索引
                    - 调用引擎接口去表数据第一行，如果不是跳过，否则这行存在结果集中
                    - 调用引擎接口取下一行，重复相同的逻辑判断，知道最后一行
                    - 将上述遍历过程中所有满足的行组成记录集作为结果集返回给客户端
                - 有索引
                    - 调用取满足条件的第一行接口，之后循环取满足条件的下一行接口
                    - 将上述遍历过程中所有满足的行组成的记录集作为结果集返回给客户端

            慢查询日志中会有rows_examined,表示这个语句执行中扫描来多少行，这个数据是执行器每次调用引擎获取数据时的累加。
            有些场景，执行器调用一次，引擎内部会扫描多行，因此引擎扫描行数和rows_examined并不是完全相等的

    - 存储引擎层

### 一条更新语句是如何执行的
流程和查询一致，只是多了个日志模块的调用，redo log（重做日志） ，binlog（归档日志），类似于酒店消费记录，一个用于速记（xxx消费了多少钱）的黑板，一个用于最后的对账（xxx最终消费了多少钱）的账本，配合使用就是mysql的wal技术（write-ahead logging）,先写日志，在写磁盘。

- 日志模块
为了使数据库在发生异常重启后，之前提交的记录不会丢失，引入了crash-safe 概念，`todoing`不是很懂

    - redo log（innodb引擎独有）
        有更新，innodb引擎会把记录写入redo log中，并更新到内存。
        写入到磁盘中，innodb会在空闲时或者redo满了时。
        redo log是循环写的，空间固定会用完，write pos 和checkpoint
    - binlog
    mysql server层的日志模块，所有引擎都可以使用
    binglog 是追加写的，写到一定大小会切换到下一个，不会覆盖

- 执行流程
    - 执行器根据where条件找引擎获取到数据，读入内存
    - 执行器根据set更新该数据后再次调用引擎接口写入数据
    - 引擎将更新后的数据写入内存，同时写入redo log中并且redo log处于prepare状态，并告知执行器，写完，可以提交事物。
    - 执行器生成该操作的binlog，并写入磁盘
    - 执行器调用引擎的提交事物接口，引擎把刚刚写入的redo log 改成commit状态，更新完成
    ```
    将redo log拆两个步骤，prepare和commit就是两阶段提交
    ```

- 为什么要两阶段提交
为了让两份日志之间的逻辑一致。
假设初始：id=2的value=0，更新value=1
    - 先写redo log，再写binlog
    redo log写入后，系统奔溃，数据回复后，value=1，但是binglog没有记录到，此时binlog相当于少记录了一条，此时用binlog恢复后value=0，原来value=1，不一致
    - 先写binlog在写redo log
    binlog写入后，系统奔溃，由于redo log 还没有写，恢复后，这个事务无效，value=0，不变，但是binlog里有value=1的记录，后面用binlog来恢复，value=1，与原来的值不一致

- 设置
innodb_flush_log_at_trx_commit=1，每次事务的redo log都持久化到磁盘，可以保存系统重启后数据不丢失
sync_binlog=1，每次事务的binlog都持久化到磁盘，保证系统重启后binlog不丢失

### 事务隔离
事务的隔离级别其实都是对于读数据的定义。
acid（原子性【atomicity】，一致性【consistency】，隔离性【isolation】，持久性【durability】）
- sql隔离有四个级别
    - 读未提交（一个事务中有修改，还未提交，其他事务已经可以看到[read uncommitted]），基本不用

    - 读已提交（一个事务中有修改，提交后，其他事务才可以看到，副本（视图，快照）会在每个sql执行时创建[read committed]）

    - 可重复度（一个事务中多次读取都是相同的结果，副本（视图，快照）会在开始时就创建[repeatable read]）

    - 串行（同一行记录，读会加锁，写也会加锁，当读写锁有冲突时，后访问的事务必须等待前一个事务执行完成才能继续[serialiable]）

这四个并行性能依次降低，安全性依次增加
通过命令`show variable like 'transaction_isolation'`来查看隔离级别
通过命令`select @@autocommit;#查看autocommit变量设置，set autocommit = 0;#关闭自动事务提交`

- 多个事务同时执行可能会出现幻读，胀读，不可重复读
    - 脏读，事务a读取了事务b未提交的数据，事务b回滚，事务a会出现错误
        - 出现在读未提交
    - 不可重复读，a事务先后执行了两个相同的查询，但是结果不一致，因为该数据被b事务修改并提交，导致a事务先后相同查询结果不一致
        - 出现在读未提交，读提交
    -  幻读，a事务修改具有摸个相同属性的批量数据，b事务此时写入该属性的一条新数据，a事务查询修改发现有一条数据没有被更改，出现幻觉
        - 出现在读未提交，读提交，可重复读
    - 更新丢失，a，b事务同时读取一条数据，a先修改并提交，b也修改（不知道a修改过），b提交后覆盖a事务修改。
        -出现在读未提交，读提交

- 事务隔离实现
每条记录在更新时都会同时记录一条回滚操作。记录上的最新值，通过回滚操作得到前一个状态值
同一个记录在系统里可以存在多个版本，就是数据库的多版本并发控制（mvcc）

    - 回滚日志
    回滚日志会在没有事务在用到的时候删除，所有尽量不要有长事务，长事务意味着系统里会存很多老的事务视图(快照)，由于这些事务随时可能会访问数据库的任何数据，所有这个事务在提交前m，数据库里他可能用的回滚记录都会保留，这就会占用存储空间。

    可通过命令查询大于60秒的长事务`select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60`

    - 事务启动
        - 显式启动
            begin或start transaction，提交commit，回滚rollback
        - set auto commit=0,关掉自动提交。
            执行一次select 事务就启动，不会自动提交，直到commit或rollback或断开连接

- 锁
读锁（共享），写锁（排他，未完成前，其他的读锁和写锁等待）
表锁，行锁，死锁（innodb 处理死锁方法，将持有最少行级排它锁的事务回滚）
    - 锁协议
        事务分两个阶段，加锁和解锁
        - 事务开始，读申请共享锁（其他事务可以申请共享锁，不能申请排他锁），更新申请写锁（其他事务不能获得锁，等待中）
        - 事务提交时，解锁
    - 主键和索引对锁影响
        1:innodb 默认行级锁,如果查询不使用索引或明确主键，为 表锁。
        2:当引擎对表加锁时，mysql server会在过滤条件时，释放不满足条件的锁。保证最后只有满足条件的数据加锁，但是加锁过程还是在的，因此会发生并未被修改的数据被加锁。

        - 有主键，有数据，行锁
        - 有主键，没有数据，没有锁
        - 无主键，表锁
        - 主键不明确，表锁
        - 有索引，有数据，行锁
        - 有索引，没数据，没有锁

- innodb引擎加锁机制
mvcc让数据变的可以重复读，那么有可能读到的里是历史数据，不是当前数据。对于数据失效性要求高的可能会出现问题。mysql为了提升并发性，减少锁处理时间，引入了快照读。
使 select 这样的可以不加锁，直接读快照。而 insert update delete 等当前读，为了解决当前读出现幻读引入了next-key锁。
    - 读可以分为两种
        - 快照读：读取历史数据的方式。 select
        - 当前读：读取和处理数据库当前版本数据的方式，需要加锁。 select .... for update insert update delete
    - mvcc
        mvcc只是个协议，没有固定的实现规范，每个数据库实现的都可以不一样，mvcc只在repeatable read 和read commited隔离级别有效， innodb的实现算不上真正实现了mvcc，mvcc本质使用乐观锁实现多版本共存，mysql使用了两段锁协议，本质是锁，乐观锁本质是消除锁，两者矛盾，故理想的mvcc是很难实现的。innodb只是借了mvcc名字实现了非堵塞的读而已。
    - mvcc原理
        每次开启一个事务都会生成一个新的事务版本号，且为递增。通过保持时间快照和读取快照来实现当同一个事务相同一个操作始终能看到相同数据，反之意味着同一时刻不同事务看到的数据可能不一致。
    - mvcc 特征
        每行数据都有一个版本，每次数据更新都同时更新版本号。 修改是copy出当前数据，各个事务之间不干扰。 保存时比较版本号，如果成功（commit），则覆盖记录，否则放弃copy修改（rollback）
    - innodb对于mvcc的实现策略
        每行数据有两个单独隐藏的列，当前行创建时的版本号，何时删除的版本号或过期的版本号（可能为空），存的版本号并不是时间戳，而是事务的版本号，每开启一个事务，事务会使用该版本号作为该事务自己的版本号，同时版本号开始递增。当事务进行curd时通过版本号的比较来达到数据版本控制的目的
    - 具体操作
        - select 读取创建版本号 <= 当前事务版本号，删除版本号为空或 >当前事务版本号
        - insert 保存当前事务版本号为行的创建版本号
        - delete 保存当前事务版本号为行的删除版本号
        - update 插入一条记录，同时保存当前事务版本号为行创建版本号，当前事务版本号为行删除版本号。

        保存这两个额外的版本号，可大部分操作不需要加锁，减少锁的使用。读数据操作很简单，性能更好，并且也能保证只读取到符合条件的行，也只锁住必要的行。不足之处在于需要额外的空间，以及更多的行检查操作和一些额外的维护工作。

    - gap 间隙锁
        - 如果查询字段使用了索引，如多条数据 user_id分别为 10，13，17，19 此时更新符合wehre 条件 user_id = 13的数据，innodb会将数据分区，无穷小-10，10-13，13-17，17-19，19到无穷大，此时会锁住10-13和13-17这个区间使数据无法插入。
        - 如果查询条件没有使用索引会给全表加锁，同时他不会向提交读隔离级别mysql server会过滤解除不满足查询查询的数据锁，除非事务提交，否则其他事务无法插入任何数据。
    - next-key锁
        - Next-Key锁为行锁和Gap的合并。 行锁可以防止不同事务版本的数据修改提交时造成数据冲突， gap防止别的事务新增， 所以next-key解决了rr级别的隔离出现的幻读问题。
