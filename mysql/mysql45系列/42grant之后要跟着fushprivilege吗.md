## 42|grant之后要跟着fush privilege吗？

> grant用来给用户赋权，主要看flush privilege和grant都做了什么事情

```mysql
create user 'ua'@'%' identified by 'pa';
```

创建一个用户 'ua'@'%'，密码时pa。

在MySQL里，用户名user+地址host才表示一个用户。

这条命令做了两个动作：

1. 磁盘上，往mysql.user表里面插入一行，由于没有指定权限，所以这行数据上所有表示权限的字段都是N；
2. 内存里，往数组acl_user里插入一个acl_user对象，这个对象的access字段为0；



### 全局权限

全局权限，作用于整个MySQL实例，这些权限信息保存在mysql库的user表里。

```mysql
grant all privileges on *.* to 'ua'@'%';
```

两个动作：

1. 磁盘上，将mysql表里，用户'ua'@'%'这一行所有表示权限的字段的值都修改为‘Y’；
2. 内存里，从数组acl_users中找到这个用户对应的对象，将access值（权限位）修改位二进制的“全1”。

如果有新的客户端使用ua登录成功，MySQL会为新的连接维护一个线程对象，然后从acl_access数组里查到这个用户的权限，并将权限值拷贝到这个线程对象中。之后在这个连接中执行语句，所有关于全局权限的判断，都直接使用线程对象内部保存的权限位。

1. grant命令对于全局权限，同时更新了磁盘和内存。命令完成后，接下来新创建的连接会使用新的权限；
2. 对于一个已经存在的连接，它的全局权限不受grant命令的影响。

生产环境合理控制权限范围。

回收权限：

```mysql
revoke all privileges on *.* from 'ua'@'%';
```

1. 磁盘上，将mysqluser表里，用户'ua'@'%'的权限字段都改为N；
2. 内存里，数组acl_access对应的对象，将access的值修改为0；



### db权限

> 除了全局权限，MySQL也支持库级别 的权限定义。

```mysql
grant all privileges on db1.* to 'ua'@'%';
```

基于库的权限记录保存在mysql.db中，在内存里则保存在数组acl_dbs中。

1. 磁盘上，往mysql.db表中插入一行记录，所有权限字段设置为Y；
2. 内存里，增加一个对象到数组acl_dbs中，这个对象的权限为全1；

每次判断一个用户对一个数据库读写权限的时候，都需要遍历依次acl_dbs数组，根据user、host和db找到匹配对象，根据权限位来判断。

grant修改db权限的时候，也是同时对磁盘和内存生效的。所有线程判断都在acl_dbs数组，对已经存在的连接，会立刻受到影响。



### 表权限和列权限

> MySQL支持更细粒度的表权限和列权限。其中，表权限定义存放在表mysql.table_priv中，列权限定义存放在表mysql.column_priv中。这两类权限，组合起来存放在内存的hash结构column_priv_hash中。

```mysql
create table db1.t1(id int, a int);
grant all privileges on db1.t1 to 'ua'@'%' with grant option;
grant select(id), insert(id,a) on mydb.mytbl to 'ua'@'%' with grant option;
```

每次grant的时候，都会修改数据表和内存的hash结构，跟db权限一样，对这两类权限操作，也会马上影响到已经存在的连接。

flush privileges命令会情况acl_users数组，然后从mysql.user表中读取数据重新加载，重新构造一个acl_user数组。同样地，对于db权限，表权限，列权限也时一样的原理。

正常情况下，grant命令之后，没有必要跟着执行flush privileges命令。



### flush privileges使用场景

> 当数据表中的权限数据跟内存中的权限数据不一致的时候，flush privileges语句可以用来重建内存数据，打到一致的状态。

这种不一致往往是不规范操作导致的，比如删除mysql.user表的某个用户，仍然可以连接进来，因为内存中acl_users数组中还有这个用户。



### 小结

- 全局权限，mysql.user表，acl_users数组；新建连接生效，已存在连接无影响；
- db权限，mysql.db表，acl_dbs数组，影响已建立的连接；
- 表权限，列权限，mysql.table_priv，mysql.column_priv表，column _priv_hash结构中；影响已建立的连接。
- 正常情况下，grant命令之后，不需要flush previliges命令；
- 不规范操作才要。比如删除mysql.user表里面信息。









