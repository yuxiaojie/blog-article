#### 开通阿里云SLB

在阿里云上，目前已经找不到vip的申请页面，查了下现在是客户有一定的量之后，工单联系客服才可以开通，所以为了实现高可用，现在使用阿里云SLB的TCP四层代理实现，阿里云SLB的四层反向代理其实也是通过 LVS +  keepalived 的方式实现 ，与自己搭建的方式差不多，就是 LVS 模式没有选择的余地

阿里云SLB如果是内网下反向代理，费用会比公网的费用低很多，所以只是内网访问实例类型就选择私网

1.png
![](https://user-gold-cdn.xitu.io/2019/9/23/16d5da9e45f40d61?w=2694&h=1700&f=png&s=426460)



#### 服务器准备

主服务器：172.19.19.35

备用服务器：172.19.19.39

两台服务器开放端口 **3366**，启用防火墙的话需要打开这两个端口，如果是阿里云的安全组的话，需要配置对应的安全策略

SLB 配置监听端口 **3306**，后端协议为 **TCP**，后端服务器类型为 **主备服务器组**，然后添加一个信息主备服务器组，然后主服务器为 172.19.19.35 端口 3366，备用服务器 172.19.19.39 端口 3366，选中下一步添加成功

2.png
![](https://user-gold-cdn.xitu.io/2019/9/23/16d5daa0ce3e2636?w=2832&h=1716&f=png&s=297063)



#### 编写 mysql 5.7 dockerfile

创建一个项目目录，然后在目录下常见一个 **dockerfile**，这里 dockerfile 的主要作用是创建了一个初始化的root密码及自定义 mysql 的配置参数 my.cnf 文件，依赖 mysql 官方镜像 5.7.26

```dockerfile
FROM mysql:5.7.26

COPY my.cnf /etc/mysql/conf.d

ENV MYSQL_ROOT_PASSWORD=yourPassword

CMD ["mysqld"]
```



然后也是通过 docker-compose 来保存启动 docker 容器的命令，也在当前目录下创建 **docker-compose.yml** 文件，主要配置了编译镜像的 context，对 mysql 服务对外的端口及持久化目录做了自定义

```yaml
version: '2'
services:
  mysql:
    build:
      context: ./
    ports:
      - "3366:3306"
    volumes:
      - "/data/mysql-test:/var/lib/mysql"
    restart: always
```



#### mysql 基本配置 

在当前目录下创建 my.cnf 对 mysql 服务参数进行配置，双主配置起始就是两台 mysql 服务器互为主从，所以每台数据库服务器都有完整的数据，也可以独立支撑业务，这里有个问题就是，每台服务器自增id的步长一样，起始值不一样，这样可以产生不一样的 id，也避免可能的id冲突

**auto-increment-offset** 就是 mysql 自增id起始值，主服务器配置 1， 备用服务器配置为2

**server-id** 是服务唯一标识，主服务器配置为035，备用服务器配置为039

这里额外开启了mysql 5.6 的一个新特性 GTID，这样在数据同步的时候，每个服务器都会使用一个全局的id，提高 mysql 同步数据时通过 GTID 来找点，尤其是复杂复制拓扑下减少数据不一致的风险

```ini
[mysqld]
########basic settings########

server-id = 039
port = 3306
max_connections = 800
max_connect_errors = 1000
log_bin = bin.log
sync_binlog = 1
expire_logs_days = 30
slow_query_log = 1
slow_query_log_file = slow.log

# 主键自增是步长为 2
auto-increment-increment=2

# 主键自增id起始为1
auto-increment-offset=1

# gtid 相关配置
log-slave-updates=ON
enforce-gtid-consistency=ON
gtid_mode=ON

# 数据同步时忽略以下数据库
binlog-ignore-db=information_schema
binlog-ignore-db=mysql
binlog-ignore-db=sys
binlog-ignore-db=performance_schema

character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log_error=/var/lib/mysql/mysql.err

sql_mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

然后使用命令 docker-compose up -d 即可自动安装镜像并启动



#### mysql 双主配置

在主服务器下使用命令连接数据库服务器，

```shell
[root@izum2vz ~]# mysql -uroot -pqwer\`1234 -P3366 -h127.0.0.1
Warning: Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1116910
Server version: 5.7.26-log MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> grant replication slave on *.* to 'repl'@'172.19.19.39' identified by 'qwer`1234';

mysql> flush privileges;

mysql> reset master;
```

这里首先创建一个用于进行同步操作的用户，限定在指定ip 172.19.19.39 中使用，然后刷新下权限，而在备用服务器上的处理也是一样的，就是用户的限定ip 指定为主服务的 ip 172.19.19.35，这里需要先把两边的账号都先创建好



主服务器配置从 master 地址用于同步数据

```shell
mysql> change master to master_host='172.19.19.39',master_port=3366,master_user='repl',master_password='qwer`1234',MASTER_AUTO_POSITION=1;

mysql> reset slave;

mysql> start slave;

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.19.19.39
                  Master_User: repl
                  Master_Port: 3366
                Connect_Retry: 60
              Master_Log_File: bin.000002
          Read_Master_Log_Pos: 8965
               Relay_Log_File: 537f2ddb3512-relay-bin.000003
                Relay_Log_Pos: 1008
        Relay_Master_Log_File: bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 8965
              Relay_Log_Space: 1222
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 39
                  Master_UUID: b1a3ad4a-dab7-11e9-b1ce-0242ac130002
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: b1a3ad4a-dab7-11e9-b1ce-0242ac130002:1-2
            Executed_Gtid_Set: 4067e27e-dab6-11e9-81da-0242ac1f0002:1-479,
b1a3ad4a-dab7-11e9-b1ce-0242ac130002:1-2
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.04 sec)

ERROR:
No query specified

mysql>
```

这里只有 Slave_IO_Running 和 Slave_SQL_Running 都为 Yes 的时候，才表示 slave模式运行成功

备用服务器的配置基本都是一样，只是将目标的 master地址替换为 主服务器地址 172.19.19.35

**show slave status\G;** 命令可以纵向显示 slave 状态信息，如果没有成功启动，可以查看错误信息



#### 导入数据

两边 slave 都配置成功后，在主数据库创建需要测试的数据库地址，然后添加一个这个数据库访问的用户，然后将初始化的数据库结构及数据的 sql 导入，这些所有的修改都会被同步到备用服务器

**注意：**

阿里云的SLB (172.19.19.40)  配置的后端服务器(即 172.19.19.39 和 172.19.19.35 两台主机) 不能直接访问 172.19.19.40 这个地址，会出现路由问题导致连接不上，需要这个网段的另外一台不在 SLB 反向代理后端的服务器才能正常访问，工单咨询客服 4层TCP透传就是有这样的问题，暂时也没有其他的方式



#### 测试

在 172.19.19.36 这台机器上，启动一个简单的 http服务器访问数据库 172.19.19.40，两台数据库服务器均数据正常写入

将 172.19.19.35 容器关闭，然后访问服务，会出现几次请求失败，然后会 http服务器 会重新连接数据库，写入正常

将 172.19.19.35 容器重新启动，然后本地连接 mysql 服务器重新开启 slave 模式

```shell
mysql> change master to master_host='172.19.19.39',master_port=3366,master_user='repl',master_password='qwer`1234',MASTER_AUTO_POSITION=1;

mysql> reset slave;

mysql> start slave;
```

这个时候再去查看数据库，也已经将 172.19.19.39 的数据同步至数据库中再次恢复一致