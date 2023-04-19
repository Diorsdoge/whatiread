## 40|insert语句的锁为什么这么多

> 有些insert语句属于特殊情况，执行过程中需要给其他资源加锁，或者无法再申请到自增id以后就立马释放自增锁

### insert ...select 语句

```mysql
create table t (
	id int not null auto_increment,
    c int default null,
    d int default null,
    primary key id,
    unique key c(c)
)engine=innodb;

insert into t values(null, 1,1);
insert into t values(null, 2,2);
insert into t values(null, 3,3);
insert into t values(null, 4,4);

create table t2 like t;
```

在rr级别，binlog_format=statement执行：

```mysql
insert into t2(c,d) select c,d from t;
```

需要对t所有的行和间隙加锁。

| session A                        | session B                              |
| -------------------------------- | -------------------------------------- |
| insert into t values(-1, -1,-1); | insert into t2(c,d) select c,d from t; |

如果session B先执行，由于这个语句对表t主键索引加了(-∞,1]这个next-key lock，会在语句执行完之后，才允许执行session A的insert语句。

但如果没有锁的话，就可能出现session B的insert语句先执行，但是后写入binlog的情况，在binlog_format=statement的情况下，这个语句到了备库执行，就会把id=-1这一行也写到t2中，导致主从不一致。



### insert 循环写入

```mysql
insert into t2(c,d) (select c+1, d from t force index(c) order by c desc limit 1);
```

这个语句的加锁范围，就是表t索引上的(3,4]和（4，suprenum]这两个next-key lock，以及主键上id=4这一行。

这样只需要扫描一行。

但是

```mysql
insert into t(c,d) (select c+1, d from t force index(c) order by c desc limit 1);
```

需要扫描5行，且看到using temporary字样，表示语句用到临时表。写入过程中需要把t内容读出来，写入临时表。

1. 创建临时表，表里面有两个字段c和d；
2. 按照索引c扫描表t，依次取c=4，3，2，1，然后回表，读到c，d的值，写入临时表；
3. limit 1，只取了临时表的第一行，再插入到t中；

也就是说，这个语句会导致在表t上做去按表扫描，并且会给索引c上的所有间隙都加上共享的nex-key lock。所以，这个语句执行期间，其他事务不能再这个表上插入数据。



### insert 唯一键冲突

| session A                                                    | session B                              |
| ------------------------------------------------------------ | -------------------------------------- |
| insert into t values(10,10,10)                               |                                        |
| begin;insert into t values(11,10,10);(duplicate entry 10 for key c) |                                        |
|                                                              | insert into t values(12,9,9);(blocked) |

session B被block了。session A在冲突的索引上加了锁，持有索引c的(5,10]next-key共享锁。

避免这一行被别的事务删掉。



#### insert into ... on duplicate key update

如果改写成

```mysql
insert into t values(11,10,10) on duplicate key update d=100;
```

的话，就会给索引c上(5,10]加上一个排他的next-key lock。

insert into ... on duplicate key update 这个语义的逻辑是，插入一行数据，如果碰到唯一键约束，就执行后面的更新语句；



### 小结

- insert ... select， 小心binlog statement模式下，主从不一致的情况；
- insert select 同一张表，可能触发临时表，造成循环写入，需要引入临时表来优化；
- insert 出现唯一键冲突，会在冲突的唯一值上加入 next-key lock 共享锁，遇到这种情况要尽快提交，避免加锁时间过长。

