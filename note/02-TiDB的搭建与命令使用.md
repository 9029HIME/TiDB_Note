# TiDB的本地搭建（伪集群）

下载TiUP

```bash
root@kjg-PC:~# curl --proto "=https" --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 7088k  100 7088k    0     0  1184k      0  0:00:05  0:00:05 --:--:-- 1646k
WARN: adding root certificate via internet: https://tiup-mirrors.pingcap.com/root.json
You can revoke this by remove /root/.tiup/bin/7b8e153f2e2d0928.root.json
Successfully set mirror to https://tiup-mirrors.pingcap.com
Detected shell: bash
Shell profile:  /root/.bashrc
/root/.bashrc has been modified to add tiup to PATH
open a new terminal or source /root/.bashrc to use it
Installed path: /root/.tiup/bin/tiup
===============================================
Have a try:     tiup playground
===============================================
```

通过TiUP指定TiDB Server、TiKV Server、PD的实例数，在本地搭建集群：

```bash
# 添加tiup的命令到环境变量
root@kjg-PC:~# source /root/.bashrc
# 这里指定3个TiDB Server，3个TiKV Server，3个PD。
root@kjg-PC:~# tiup playground v6.1.0 --db 3 --pd 3 --kv 3 --host 192.168.120.161
tiup is checking updates for component playground ...
A new version of playground is available:
The latest version:         v1.11.3
Local installed version:    
Update current component:   tiup update playground
Update all components:      tiup update --all
The component `playground` version  is not installed; downloading from repository.

download https://tiup-mirrors.pingcap.com/playground-v1.11.3-linux-amd64.tar.gz 7.44 MiB / 7.44 MiB 100.00% 37.07 MiB/s                  
Starting component `playground`: /root/.tiup/components/playground/v1.11.3/tiup-playground v6.1.0 --db 3 --pd 3 --kv 3 --host 192.168.120.161
Playground Bootstrapping...
Start pd instance:v6.1.0
The component `pd` version v6.1.0 is not installed; downloading from repository.
download https://tiup-mirrors.pingcap.com/pd-v6.1.0-linux-amd64.tar.gz 43.25 MiB / 43.25 MiB 100.00% 26.67 MiB/s                         
Start pd instance:v6.1.0
Start pd instance:v6.1.0
Start tikv instance:v6.1.0
The component `tikv` version v6.1.0 is not installed; downloading from repository.
download https://tiup-mirrors.pingcap.com/tikv-v6.1.0-linux-amd64.tar.gz 199.70 MiB / 199.70 MiB 100.00% 28.36 MiB/s                     
Start tikv instance:v6.1.0
Start tikv instance:v6.1.0
Start tidb instance:v6.1.0
The component `tidb` version v6.1.0 is not installed; downloading from repository.
download https://tiup-mirrors.pingcap.com/tidb-v6.1.0-linux-amd64.tar.gz 51.48 MiB / 51.48 MiB 100.00% 20.97 MiB/s                       
Start tidb instance:v6.1.0
Start tidb instance:v6.1.0
Waiting for tidb instances ready
192.168.120.161:4000 ... Done
192.168.120.161:4001 ... Done
192.168.120.161:4002 ... Done
The component `prometheus` version v6.1.0 is not installed; downloading from repository.
download https://tiup-mirrors.pingcap.com/prometheus-v6.1.0-linux-amd64.tar.gz 87.63 MiB / 87.63 MiB 100.00% 29.87 MiB/s                 
download https://tiup-mirrors.pingcap.com/grafana-v6.1.0-linux-amd64.tar.gz 50.08 MiB / 50.08 MiB 100.00% 30.14 MiB/s                    
Start tiflash instance:v6.1.0
The component `tiflash` version v6.1.0 is not installed; downloading from repository.
download https://tiup-mirrors.pingcap.com/tiflash-v6.1.0-linux-amd64.tar.gz 184.46 MiB / 184.46 MiB 100.00% 28.38 MiB/s                  
Waiting for tiflash instances ready
192.168.120.161:3930 ... Done
CLUSTER START SUCCESSFULLY, Enjoy it ^-^
To connect TiDB: mysql --comments --host 192.168.120.161 --port 4000 -u root -p (no password)
To connect TiDB: mysql --comments --host 192.168.120.161 --port 4002 -u root -p (no password)
To connect TiDB: mysql --comments --host 192.168.120.161 --port 4001 -u root -p (no password)
To view the dashboard: http://192.168.120.161:42641/dashboard
PD client endpoints: [192.168.120.161:42641 192.168.120.161:42561 192.168.120.161:38151]
To view the Prometheus: http://192.168.120.161:9090
To view the Grafana: http://192.168.120.161:3000 
```

最终的架构图如下：

![01](02-TiDB的搭建与命令使用.assets/01.png)

# TiDB本地集群的使用

## 连接与关闭

TiDB支持MySQL的协议，因此直接用MySQL客户端连接即可，刚才创建的本地集群没有密码，只要随机连上其中一个TiDB Server即可：

```bash
root@kjg-PC:~# mysql --host 192.168.120.161 --port 4000 -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 407
Server version: 5.7.25-TiDB-v6.1.0 TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| INFORMATION_SCHEMA |
| METRICS_SCHEMA     |
| PERFORMANCE_SCHEMA |
| mysql              |
| test               |
+--------------------+
5 rows in set (0.00 sec)
```

新建一张表试试：

```sql
mysql> use test;
Database changed
mysql> CREATE TABLE `test`.t_admin (`id` bigint(20) not null auto_increment comment '主键',`username` varchar(50) comment '用户名',PRIMARY KEY(`id`) USING BTREE);
Query OK, 0 rows affected (0.53 sec)

mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| t_admin        |
+----------------+
1 row in set (0.00 sec)
```

此时可以把TIDB当成一个MySQL来使用，不过通过TiUP创建的TiDB集群有个缺陷，关闭服务后，数据会丢失：

```bash
^CGot signal interrupt (Component: playground ; PID: 35906)                                                                              
Playground receive signal:  interrupt
Wait tiflash(36659) to quit...
Wait prometheus(36562) to quit...
Wait grafana(36649) to quit...
Wait ng-monitoring(36563) to quit...
Grafana quit
prometheus quit
ng-monitoring quit
Wait tidb(35940) to quit...
tiflash quit
tidb quit
Wait tidb(35954) to quit...
tidb quit
Wait tikv(35931) to quit...
tikv quit
Wait tikv(35935) to quit...
tikv quit
Wait pd(35917) to quit...
pd quit
Wait pd(35920) to quit...
pd quit


# 重启TiDB后我再连接，发现刚才创建的t_admin表不在了。
kjg@kjg-PC:~$ mysql --host 192.168.120.161 --port 4000 -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 407
Server version: 5.7.25-TiDB-v6.1.0 TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use test;
Database changed
mysql> show tables;
Empty set (0.00 sec)
```

## 
