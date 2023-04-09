## 22|MySQL有哪些“饮鸩止渴”提高性能得方法？

作者想要将一些“先让业务跑起来再说”得场景。



### 短连接风暴

正常短连接模式：连接数据库后，执行很少SQL就断开。断开重连，MySQL建立连接的过程成本其实很高的。三次握手外，还需要做登录权限判断和获得这个连接的数据读写权限。

max_connections 参数，用来控制一个MySQL实例同时存在的连接数上线，超过这个值，系统会拒绝接下来的请求，并报错“too many connections”。业务看来，就是数据库不可用。

#### 一种方法：先处理掉那些占着连接但是不工作的线程。

max_connections的计算，并不是看谁在running，只要连接着就占用一个计位数。可以通过kill connections主动踢掉。设置wait_timeout表示空闲多少秒，直接断开。但需要注意，kill掉sleep的线程可能是有损的。比如A事务只是没有提交，但已经跑完了其他语句，被kill掉之后会回滚。

如果是连接数过多，可以优先断开事务外空闲太久的连接，如果不够，再考虑事务内空闲太久的连接。

#### 第二种方法：减少连接过程消耗

可以跳过连接鉴权。使用skip-grant-tables参数重新启动，会跳过鉴权。但这样奉献极高，基本上不满足安全部门需求，分分钟不合规。不到万不得已，切忌使用。



### 慢查询性能问题

慢查询，大体有三种可能：

1. 索引没设好；
2. SQL语句没写好；
3. MySQL选错了索引。

索引没设计好，一般通过紧急创建索引解决，MySQL5.6之后，创建索引都支持online DDL了。

最好再备库先加，再主从切换，再在新备库上加：

1. 备库B上执行set sql_log_bin=off，不写binlog，添加索引，因为binlog太大；
2. 主从切换；
3. 备库A上执行set sql_log_bin=off，不写binlog，添加索引。

语句没写好，例如前面：

select * from t where id+1=100000; 不走索引，改写为id=1000000-1；在语句写错的情况下，增加一个语句改写规则：

```mysql
insert into query_rewrite.rewrite_rules(pattern, replacement, pattern_database) values ("select * from t where id+1=?", "select * from t where id=?-=1","db1");
call query_rewrite.flush_rewrite_rules();
```

选错索引，可以使用force index，解决问题。

尽量避免：

1. 上线前，在测试环境，把慢日志打开，并把long_query_time设置成0；
2. 测试数据库操作，做一次回归测试；
3. 注意rows_examined字段是否与预期一致；

可以使用工具pt-query-digest。



### QPS 突增问题

突然出现高峰，或者业务程序出现bug，可能导致某个语句的QPS飙升。

1. 业务bug导致，直接把业务从白名单中去掉；
2. 删除引发QPS的账户，再断开所有连接；
3. 如果新业务跟主体业务是在一起的，把压力最大的SQL语句给直接重写为”select 1“；



### 小结

- 短连接风暴：kill 掉一些connections，sleep中已提交的事务优先；减少连接消耗，跳过鉴权；
- 慢查询：索引没整好，语句没整好，MySQL选错了索引；
- QPS突增：禁新业务IP；删新业务账户，kill 所有连接；重写最重SQL为”select 1“