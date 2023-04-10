## 23|MySQL是怎么保证数据不丢的？

WAL的机制（第2，9，12，15篇），结论：只要redo log和binlog保证持久化到硬盘，就能确保MySQL异常重启，数据可以恢复；

### binlog的写入机制

事务执行过程中，先把日志写道binlog cache，事务提交的时候再把binlog cache写到binlog文件中。一个事务的binlog是不能被拆开的，因此不管这个事务由多大，也要确保一次性写入。

系统给binlog cache分配了一片内存，每个线程一个，参数binlog_cache_size用于控制单个线程内binlog cache所占内存大小。如果超过了这个参数规定的大小，就要暂存到磁盘（内存+临时文件）。

事务提交的时候，执行器把binlog cache里完整的事务写入binlog文件，并清空binlog cache。过程分为了write和fsync两部分；

每个线程有自己的binlog cache，但共用一份binlog文件。

- write：指的是把日志写入到文件系统的page cache，并没有把数据持久化到磁盘，速度很快；
- fsync：将数据持久化到硬盘。一般认为fsync才占磁盘的IOPS。

write和fsync的时机模式有sync_binlog控制的：

1. sync_binlog=0,表示每次提交事务都只write，不fsync；
2. sync_binlog=1的时候，表此每次提交事务都会执行fsync；
3. sync_binlog>N(N>1)的时候，表示每次事务都会write，但积累N个事务后才fsync。

所以，当IO瓶颈场景里，可以将sync_binlog设置为一个比较大的值，可以提升性能。比较常见的是设为100-1000之间的值。但，对应的风险是，如果主机一场重启，会丢失最近N个事务的binlog。



### redo log的写入机制

生成的redo log 要先写入redo log buffer。

redo log可能存在三种状态：

1. redo log buffer中，物理上实在MySQL进程内存中；
2. 写到磁盘write，但未持久化fsync，物理上实在page cache里；
3. 持久化到磁盘，对应的是hard disk。

为了控制redo log的写入策略，InnoDB提供了innodb_flush_log_at_trx_commit参数：

1. 0：表示每次事务提交时，都只把redo log留在redo log buffer中；
2. 1：每次提交事务都持久化到磁盘；
3. 2：每次提交事务都写到page cache；

InnoDB有个后台线程，每隔1s，就会把redo log buffer中的日志，调用write被后台线程一起写到文件系统的page cache，再fsync到磁盘。

执行过程中的redo log也写在redo log buffer中，这些redo log 也会被后台进程一起持久化到磁盘。这么看来binlog cache是每个线程都由一个，redo log buffer是所有线程公用一个。

除了轮询换，还有两个场景会让未提交的事务redo log写入打磁盘：

1. redo log buffer占用的空间即将打到innodb_log_buffer_size的一半，后台线程会主动写盘；
2. 并行事务提交，顺带把redo log buffer持久化到磁盘；

MySQL的双1，就是sync_binlog和innodb_flush_at_trx_commit都设置为1.也就是说，一个事务完整提交前，需要等待两次刷盘，一次是redo log（prepare阶段），一次是binlog。

MySQL tps是每秒2万得话，那每秒就会写四万次磁盘。磁盘的能力也就2w左右，怎么实现的？

gourp commit机制。

LSN(日志逻辑序列号)，单调递增，用来对应redo log的一个个写入点。每次写入长度为length的redo log，LSN的值就会加上length。LSN也会写到InnoDB的数据页中，来确保数据月不会被多次执行重复的redo log。

三个并发事务(trx1, trx2, trx3)在prepare阶段，都写完redo log buffer，持久化到磁盘的过程，对应LSN是50、120、160。

1. trx1是第一个到达的，会被选为这组的leader；
2. 等trx1开始写盘的时候，这个组里面已经由三个事务，这时候LSN变成了160；
3. trx1去写盘的时候，带的就是LSN=160，因此等trx1返回时，所有LSN小于等于160的redo log，都已经被全部持久化到磁盘；
4. 这时候trx2和trx3就可以直接返回的。

所以，一次组提交里面，组员越多，节约磁盘IOPS效果越好；但如果时单线程压测，那就只能老老实实一个事务对应一次持久化操作；

为了让一次fsync带的组员更多，MySQL的优化策略为：拖时间。MySQL为了让组提交的效果好，把redo log和binlog做fsync交叉互换，过程为：

1. redo log prepared: write;
2. binlog: write;
3. redo log prepare: fsync;
4. binlog: fsync;
5. redo log commit: write.

如果向提升binlog组提交的效果，可以通过设置binlog_group_commit_sync_delpay和binlog_group_commit_sync_no_delay_count来实现。

1. binlog_group_commit_sync_delpay 表示延迟多少微妙后才fsync；
2. binlog_group_commit_sync_no_delay_count 表示积累多少此后，才调用fsync。

这两个参数时或关系，只要有一个满足条件，都会调用fsync。

WAL的机制主要得益于两个方面：

1. redo log和binlog都是循序写；
2. 组提交机制，可以大幅度降低磁盘的IOPS消耗；

如果出现IO瓶颈：

1. 设置binlog_group_commit_sync_delpay和binlog_group_commit_sync_no_delay_count 参数，减少binlog写盘次数，可能会增加语句响应实践，但没有丢失数据风险；
2. sync_binlog大于1的值，风险是主机掉点，会丢binlog日志；
3. innodb_flush_log_at_trx_commit设置为2，风险时，主机掉电，会丢数据。



### 小结

- binlog 的写入机制： 写binlog cache，write，fsync；
- redo log的写入机制：写redo log buffer，write， fsync；
- 为提高IO，使用组fsync。核心方法是，同一段时间内事务，一起fsync。