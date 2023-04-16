## 35|join语句怎么优化？

```MYSQL
create table t1(id int primary key, a int, b int, index(a));
create table t2 like t1;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
	declare i int;
	set i=1;
	while(i<1000) do
		insert into t1 values(i,1001-i,i);
		set i=i+1;
	end while;
	
	set i=1;
	while(i<1000000) do
		insert into t2 values(i,1001-i,i);
		set i=i+1;
	end while;
	
end ;;
delimiter ;
call idata();
```



### Multi-Range Read优化

> 这个优化的目的是顺序读盘。MRR

MRR设计思路：**因为大多数的数据都是按照主键递增顺序插入得到的，我们可以认为如果按照主键递增顺序查询的话，对磁盘的都比较接近顺序读，能提升读性能。**

语句执行流程为：

1. 根据索引a，定位到满足条件的记录，将id放入read_rnd_buffer中；
2. 将read_rnd_buffer中的id进行递增排序；
3. 排序后的id数组，一次到主键id索引中查记录，并作为结果返回；

read_rnd_buffer：MySQL的随机读缓冲区。

MRR能够提升性能的核心在于：这条查询语句在索引上做的是一个范围查询，可以得到足够多的id，这样通过排序后，再去主键查询，才能体现出优势。



### Batched Key Access

MySQL 5.6之后，引入BKA算法，其实是对NLJ算法的优化。

核心：一次性多拿出一些t1的值，一起传给t2；这些值存在join_buffer中。

要使用BKA，要先设置

```mysql
set optimizer_switch='mrr=on,mrr_cost_based=off,batch_key_access=on';
```

前两个参数时启用mrr，这么作的原因是BKA算法的优化要依赖于MRR。



### BNL的性能问题

如果一个使用BNL算法的join语句，多次扫描一个冷表，而且这个语句执行时间超过1s，就会再次扫描冷表的时候，把冷表数据移到LRU的链表头部（young区）。

如果这个冷表很大，会出现：业务正常发访问数据页，没有机会进入young区，使得热点数据一直从磁盘读取，young区的数据得不到更新，数据没有被合理的淘汰。

大表join操作虽然对IO有影响，但是在语句执行结束后，对IO的影响也就结束了。但是，对buffer pool的影响是持续性的，需要依靠后续查询请求慢慢恢复内存命中率。

减少这种影响，可以考虑增大join_buffer_size的值，减少对被驱动表扫描的次数。

BNL影响的三个方面：

1. 可能会多次扫描被驱动表，占用磁盘IO资源；
2. 判断M*N次对比，大表会占用非常多的CPU资源；
3. 可能会导致Buffer Pool的热数据被淘汰，影响内存命中率。

可以把BNL算法，转换为BKA算法。

### BNL转BKA

```mysql
select * from t1 join t2 on (t1.b=t2.b);
```

如果使用BNL:

1. 把表t1所有字段取出来，存入join_buffer;
2. 扫描t2，取出每一行数据跟join_buffer的数据对比；
   - 如果不满足条件，跳过；
   - 如果满足，在判断其他条件，都满足，则作为结果集一部分，有不满足，则跳过。

解决思路：

1. 表t2中满足条件的数据放在临时表中tmp_t;
2. 给临时表tmp_t的字段b加上索引；
3. 让表t1和tmp_t作join操作；



总的来说，不论是在原表上加索引，还是用有索引的临时表，思路都是使用被驱动表上的索引，触发BKA算法。



### 扩展-hash join

> MySQL不支持hash join，但可以由业务自己实现。

实现流程大致如下：

1. select * from t1;取得t1全部数据，在业务端存入一个hash结构；
2. select * from t2 where b>=1 and b<=2000; 获取满足条件的t2行；
3. 把这2000行取到业务端，到hash结构中进行匹配，满足条件的就作为结果集的一部分。

实际上，也就是这种情况不推荐join



### 小结

- NLJ 优化，BKA
- BNL的性能问题，IO大，CPU使用多，影响内存命中
- BNL转BKA，核心：在被驱动表上建索引，也可以引入临时表，把临时表作为被驱动表
- 在业务端使用hash-join的方法来提高速度