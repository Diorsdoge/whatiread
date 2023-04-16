## 34|到底可不可以使用join？

>1. 我们DBA补让使用join，使用join有啥问题咯？
>2. 如果有两个大小不同的表做jioin，因该用哪个表做驱动表呢？

为了便于量化分析，作者还是创建了表t1和t2；

```mysql
create table t2 (
	id int(11) not null,
    a int(11) defualt null,
    b int(11) defualt null,
    primary key (id),
    key a(a)
)engin=innodb;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
	declare i int;
	set i=1;
	while(i<1000) do
		insert into t2 values(i,i,i);
		set i=i+1;
	end while;
end ;;
delimiter ;
call idata();

create table t1 like t2;
insert into t1 (select * from t2 where id<100);
```



### Index Nested-Loop Join

```mysqk
sekect * from t1 straight_join t2 on (t1.a=t2.a);
```

这个语句中，t1作为驱动表，t2是被驱动表，被驱动表字段a上有索引，join用上可这个索引：

1. 从表t1取出一行；
2. 从数据R中，取出a字段到表t2里去查找；
3. 取出表t2中满足条件的行，跟R组成一行，作为结果集的一部分；
4. 重复步骤1到3，直到t1的末尾循环；

这个过程先遍历表t1，在根据t1取出的a值去t2查找满足条件的记录，并且用上了被驱动表的索引，这个称为“Index Nested-Loop Join"，简称NLJ。

这个流程中，表t1扫描了100行，表t2也扫描了100行，总共200行；

回到问题1，能不能使用join？

假设不使用join，就会但表查询：

1. select * from t1；取出100行；
2. 循环遍历这100行，根据a在t2中取值；再把返回的结果和1中取出的结果构成结果集中的一行；

共扫描了200行，但执行了101条语句，显然还不如直接用join。

整个执行过程，假设驱动表N行，被驱动表M行，近似复杂度为：N+N*2logM；显然N对扫描行数的影响更大，因此应该让**小表来做驱动表**。

结论：

1. 使用join语句，性能比强行拆成多个单表执行的SQL语句性能更好；
2. 小表作为驱动表；

但这个的前提是**可以使用被驱动表的索引**



### Simple Nested-Loop Join

```MYSQL
sekect * from t1 straight_join t2 on (t1.a=t2.b);
```

由于表t2的字段b上没有索引，每次t2的都会走一次全表扫描。这样叫做”Simple Nested-Loop Join“。

这样总共扫描 100 * 1000 = 10万行。

如果表更大，将会更恐怖。

MySQL没有使用这个算法；



### Block Nested-Loop Join

> 简称BNL

被驱动表上没有可用的索引，算法流程是这样的

1. 把表t1的数据读入线程内存，join_buffer中，由于我们这个语句中写的是select *, 因此是把整个表t1放入内存；
2. 扫描表t2，把t2中的每一行取出来，跟join_buffer中的数据对比，满足条件的，作为结果集的一部分返回；

总共扫描1100行，t2的每一行都要做100次判断，总共判断10万次。这种比simple nested-loop join速度快很多，性能更好。

假设小表行数N，达标行数M，那么在这个算法里：

1. 扫描M+N行；
2. 内存判断次数M*N。

可以看出不管是大表还是小表作驱动表，执行耗时是一样的。

要是join_buffer放不下咋办，这个由join_buffer_size控制，默认256k。如果放不下表t1的所有数据的话，策略为分段放。

1. 扫描t1，顺序读取行放入join_buffer中，放完第88行join_buffer满了，继续第二步；
2. 扫描t2，把t2每一行取出来，跟join_buffer作对比，满足join条件，作为结果集一部分；
3. 清空join_buffer;
4. 继续扫描t1。

驱动表行数N，被驱动表行数M，分K段，K不是常数，K可以表示为 λ*N，显然这个取值范围是(0,1)

1. 扫描行数：N + λ\*N\*M
2. 内存判断：N*M次

内存判断不受选择哪个表为驱动表控制，但是扫描行数会，N小一些，整体会好些；

结论，应该让小表当驱动表。



问题1：能不能用join？

1. 如果可以使用 NLJ 算法，其实没问题的；
2. 如果不是，则尽量不要用；占用的资源量太大。

问题2：如果能用，用大表还是小表作驱动表？

总应该使用小表作驱动表。但如果是加了限定条件，比如 t2.a < 10这一类，那么t2就成了小表。

更准确的说，**在决定那个表作驱动表的时候，应该两个表按照各自的条件过滤，过滤完之后，计算参与join的各个字段总数据量，数据量小的哪个表，就是小表，作为驱动表**



### 小结

- 如果可以使用被驱动表的索引，join语句是有优势的
- 否则其实优势不大，反而给数据库增加压力
- 尽量使用小表作驱动表，小表是条件限定过滤之后小的那个”表“