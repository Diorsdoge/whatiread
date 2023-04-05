## 16|“order by”是怎么工作的？

假设表的部分定义：

```mysql
create table `t` (
	`id` int(11) not null,
    `city` varchar(16) not null,
    `name` varchar(16) not null,
    `ag`name` varchar(16) not null,e` int(11) not null,
    `addr` varchar(128) default null,
    primary key (`id`),
    key `city`(`city`)
)engine=innodb;
```

```mysql
select city, name, age from t where city="hangzhou" order by name limit 1000;
```



### 全字段排序

在这条SQL中，执行计划会走city索引，并会用到sort_buffer，表示每个线程分配得到的一块内存；通常情况下，语句的执行流程如下：

1. 初始化sort_buffer，确定放入name、city、age这三个字段；
2. 从索引city找到第一个满足city=‘hangzhou’条件的主键id；
3. 到主键id中去取出整行，取name、city、age三个字段的值，存入sort_buffer中；
4. 从索引city取下一个记录的主键id；
5. 重复步骤3、4，直到city不满足条件为止；
6. 对sort_buffer中的数据按照字段name做快速排序；
7. 按照排序结果取前1000行返回给客户端。

”按name排序“可能在内存中完成，页有可能使用外部排序，这取决于排序所需要的内存和参数sort_buffer_size。

sort_buffer_size就是MySQL为排序开辟的内存大小。如果要排序的数据量小于sort_buffer_size，则排序在内存中完成，若数据量太大，则不得不利用磁盘临时文件来辅助排序。

外部排序一般使用归并排序算法：MySQL将需要排序的数据分成number_of_tmp_files份，每一份单独排序后存在这些临时文件中。然后把这12个有序文件再合成一个有序的大文件。



### rowid排序

上面这个算法过程中，只对原表读了一边。当查询需要返回的字段很多的时候，放入sort_buffer的数据量太大，会分成很多磁盘上的临时文件，这个时候的排序性能会很差。所以单行太大，这个方法效率不太好。

修改一个参数：

```MYSQL
set max_length_for_sort_data= 16
-- max_length_for_sort_data参数是MySQL中专门控制用于排序长度的一个参数，表示单行长度超过这个值，MySQL就认为单行太大，要换一个算法
```

city、name、age这三个字段的定义总长度是36，max_length_for_sort_data为16，此时的算法为：只将name和主键id放入sort_buffer中排序。然后根据id回表取回三个字段返回给客户端。执行流程如下：

1. 初始化sort_buffer，确定放入两个字段name、id；
2. 从索引city找到第一个满足city=‘hangzhou’条件的主键id；
3. 到主键id中去取出整行，取name、id两个字段的值，存入sort_buffer中；
4. 从索引city取下一个记录的主键id；
5. 重复步骤3、4，直到city不满足条件为止；
6. 对sort_buffer中的数据按照字段name做快速排序；
7. 遍历排序结果，取前1000行，并按照id的值回表取出city、name、age三个字段返回给客户端。



### 全字段排序 VS rowid排序

MySQL的一个设计思想：如果内存够，就要多利用内存，尽量减少磁盘访问；

如果担心排序内存太小，会影响排序效率，才会采用rowid排序；正常情况下，都是走全字段排序。

MySQL中排序是一个比较高的操作，其实并不是所有的order by都需要排序操作，排序的原因其实是原来的数据都是无序的。如果能保证city这个索引上取出来的数据，天然就是按照name递增排序的，就可以不用排序。

````MYSQL
alter table t add index city_name(city, name) -- 新建联合索引
````

在这个索引下，所有name的值就一定是有序的，这时候的查询流程就变成了:

1. 从索引(city, name)找到第一个满足city='hangzhou'条件的主键id；
2. 到主键id索取出整行，取name、city、age三个字段的值；
3. 从索引(city, name)找下一个记录主键id；
4. 重复2、3步骤，直到查到1000条记录或者不满足条件为止。

可以看到这个时候是不需要排序的，也不会用到临时文件。

如果使用”覆盖索引“继续优化，即添加三个字段的索引：

```MYSQL
alter table t add index city_name_age(city, name, age);
```

这样即不会回表，流程变成了：

1. 从索引city_name_age找到第一个满足条件的记录，取出三个字段的值，作为结果集的一部分直接返回；
2. 从索引city_name_age取下一个记录，同样取出三个字段的值，作为结果集的一部分直接返回；
3. 重复2，直到查到1000条记录或者不满足条件为止。

具体情况还是需要权衡，不是所有地方都适合使用覆盖索引。



### 小结：

- 全字段排序，即把所有字段都塞到sort_buffer中排序；
- rowid排序，字段的字节总数超出设定的值，则只取需要排序的字段，和主键id进入sort_buffer排序，之后回表取出对应的字段；
- 排序使用超出sort_buffer容量，会使用磁盘的临时文件，采用归并排序；



