# 一种简单的MySQL高可用方案

两主一从+HaProxy反向代理实现高可用，从节点只用于热备份，延迟2小时同步。

## 配置文件
m1和m2的区别在于`server-id`和`auto_increment_offset`。组主从，`server-id`必须唯一（实际情况可以直接用IP末尾）。
因为是双主，为了保证自增主键不重复，主键增量为`2`，m1是单数递增，m2是双数递增。

> 这样配置后，实际主键会出现跳号

s1的配置文件，`server-id`同样需要唯一。除此之外，s1作为纯备份库，没有开启binlog日志（开启也可以，这个看实际情况）。并设为**只读**

## 配置两主一从

### 给3个节点都创建一个同步用的帐号
```shell script
# 进入docker容器
docker-compose exec m1 bash

# 登录MySQL
mysql -uroot -p123456

# 创建同步用户
grant replication slave on *.* to 'replicate'@'%' identified by '123456';

# 查看master状态
show master status\G;
```

### m1配置

m1和m2互为主从，`master_log_file`和`master_log_pos`的取值根据实际情况修改

```shell script
# 用上一步创建的同步用户和master状态，配置从节点
change master to
master_host='m2',
master_user='replicate',
master_password='123456',
master_port=3306,
master_log_file='mysql-bin.000003',
master_log_pos=447;
```

### m2配置
```shell script
change master to
master_host='m1',
master_user='replicate',
master_password='123456',
master_port=3306,
master_log_file='mysql-bin.000003',
master_log_pos=740;
```

### s1配置

s1从m1同步，延迟7200秒。某些小问题（例如直接手动修改数据库改错数据了），可以在2小时内补救。

```shell script
change master to
master_host='m1',
master_user='replicate',
master_password='123456',
master_port=3306,
master_log_file='mysql-bin.000003',
master_log_pos=740,
master_delay=7200;
```

### 开启同步并查看同步状态
```shell script
start slave;

# 查看同步状态
show slave status\G;
```

如果`Slave_IO_Running`和`Slave_SQL_Running`的值都是`Yes`，表示启动成功了
