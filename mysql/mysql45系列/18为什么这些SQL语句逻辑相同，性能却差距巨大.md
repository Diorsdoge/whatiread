## 18|为什么这些SQL语句逻辑相同，性能却差异巨大？

这里分享了三个案例。

### 案例一：条件字段函数操作

假设MySQL45先生维护了一个交易系统，其中交易记录表tradelog包含交易流水号（tradeid）、交易员id（operator）、交易时间（t_modified）

```mysql
create table tradelog (
	id int(11) not null,
    tradeid varchar(32) default null,
    operator int(11) defualt null,
    t_modified datetime default null,
    primary key(`id`),
    key `tradeid`(`tradeid`),
    key `t_modified`(`t_modified`)
)engine=innodb defualt charset=utf8mb4;
```

要查从2016年到2018年底中，统计发生在所有年份中7月份的交易记录总数：

```mysql
select count(*) from tradelog where month(t_modified)=7;
```

虽然t_modified加了索引，但实际上，它不会走索引，仍然走全表扫描：如果对字段做了函数计算，就用不上索引了，这是MySQL的规定。

如果使用t_modified='2018-07-01'，引擎是会走索引，快速定位到需要的结果，因为在这颗B+树上，同一层兄弟节点之间是有序的。

如果是计算month()函数，传入7的时候，在B+树的第一层就不知道怎么办了。

**对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器会放弃走索引树功能。**

改

```mysql
select count(*) from tradelog where  (t_modified >= '2016-7-1' and t_modified < '2016-8-1') or (t_modified >= '2017-7-1' and t_modified < '2017-8-1') or (t_modified >= '2018-7-1' and t_modified < '2018-8-1')
```

即便不破坏有序性，依然不能使用索引。例如：where id + 1=10000 不行，需要改为 where id = 10000 - 1.



### 案例二：隐式类型转换

看下面这个SQL

```mysql
select * from tradelog where tradeid=110717;
```

有索引，但还是走了全表扫描。tradeid的字段是varchar(32), 而输入的参数是整型。

在MySQL中，字符串和数字比较的话，是将字符串转换成数字。

对优化器来说，其实是：

```mysql
select * from tradelog where cast(tradeid as signed int)=110717;
```

相当于所该字段做了函数操作，使之不能使用索引。

假如换成id=‘11109’， 由于字符串会转整型，且不是对索引字段的函数，是可以走索引的。



### 案例三：隐式字符编码转换

假设还有另一个表 trade_detail

```mysql
create table trade_detail (
	id int(11) not null,
    tradeid varchar(32) default null,
    trade_step int(11) defualt null,
    step_info varchar(32) default null,
    primary key(`id`),
    key `tradeid`(`tradeid`)
)engine=innodb defualt charset=utf8;

insert into tradelog values(1, 'aaaaaaaa', 1000, now());
insert into tradelog values(2, 'aaaaaaab', 1000, now());
insert into tradelog values(3, 'aaaaaaac', 1000, now());

insert into trade_detail values(1, 'aaaaaaaa', 1, 'add');
insert into trade_detail values(2, 'aaaaaaaa', 2, 'update');
insert into trade_detail values(3, 'aaaaaaaa', 3, 'commit');
insert into trade_detail values(4, 'aaaaaaab', 1, 'add');
insert into trade_detail values(5, 'aaaaaaab', 2, 'update');
insert into trade_detail values(6, 'aaaaaaab', 3, 'update again');
insert into trade_detail values(7, 'aaaaaaab', 4, 'commit');
insert into trade_detail values(8, 'aaaaaaac', 1, 'add');
insert into trade_detail values(9, 'aaaaaaac', 2, 'update');
insert into trade_detail values(10, 'aaaaaaac', 3, 'update again');
insert into trade_detail values(11, 'aaaaaaac', 4, 'commit');
```

如果要查id=2的信息：

```mysql
select * from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=2;
```

根据执行计划可以发现：

1. 优化器会先在交易记录表tradelog上查到id=2的行，这个步骤用上了主键索引；
2. trade_detail的tradeid索引没有用上，走全表扫描。

这个写法中tradelog为驱动表，trade_detail为被驱动表，关联字段为tradeid；

其步骤为

1. 根据id在tradelog表里找到L2这一行；
2. 从L2中取出tradeid字段的值；
3. 根据trandeid值到trade_detail表中查找条件匹配的行，这个过程为通过遍历主键索引的方式，一个一个地判断tradeid的值是否匹配；

发现第三步不符合我们的预期；因为两个表使用的字符集不同，一个是utf8mb4，一个是utf8；

utf8mb4是utf8的超集，二者比较的时候，MySQL会将utf8转换成utf8mb4。在执行3的时候，需要将被驱动表中的字段一个个地转成utf8mb4，再跟驱动表做比较。等同于使用了convert(tradeid using utf8mb4) = $L2.tradeid.value 进行数据转换，又触发了原则：对索引字段做函数操作，优化器会放弃走索引树。

```mysql
select l.operator from tradelog l, trade_detail d where d.tradeid=l.tradeid and d.id=2;
```

这个语句中trade_detail成为了驱动表，可以发现被驱动表tradelog的tradeid索引被使用了，没有走全表扫描；这个时候相当于tradeid = convert($R4.trade.value using utf8mb4).

两种做法：

1. 修改字符集，改为统一的utf8mb4;`alter table trade_detail modify tradeid varchar(32) character set utf8mb4 default null` 
2. 如果数据量大，或者业务暂时不能DDL，可以使用修改SQL的方法，把函数放入参数中去。



### 小结

- 核心，索引字段上使用了函数，MySQL不会走索引的。
- 隐式类型转换，如果在索引字段，不走索引；如果在参数上，则可以走索引；
- 经常explain一下，看看执行计划