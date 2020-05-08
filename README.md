# Using Docker containers to setup ProxySQL + MySQL Replication

**ProxySQL** (https://proxysql.com/) is an open-source MySQL proxy server, meaning it serves as an intermediary between a MySQL server and the applications that access its databases. ProxySQL can improve performance by distributing traffic among a pool of multiple database servers and also improve availability by automatically failing over to a standby if one or more of the database servers fail.

## ProxySQL config file

Let's see the main definitions we have in `proxysql.cnf` config file:

* the proxySQL admin credentials and port to access `6032`.
```
admin_variables=
{
    admin_credentials="admin:admin;radmin:radmin"
    mysql_ifaces="0.0.0.0:6032"
    refresh_interval=2000
}
```

* the hostgroups and the role of which MySQL server.
```
mysql_replication_hostgroups =
(
    { writer_hostgroup=10 , reader_hostgroup=20 , comment="host groups" }
)

mysql_servers =
(
    { address="master" , port=3306 , hostgroup=10, max_connections=100 , max_replication_lag = 5 },
    { address="slave1" , port=3306 , hostgroup=20, max_connections=100 , max_replication_lag = 5 },
    { address="slave2" , port=3306 , hostgroup=20, max_connections=100 , max_replication_lag = 5 }
)
```

* MySQL user:
```
mysql_users =
(
    { username = "root" , password = "mypass" , default_hostgroup = 10 , active = 1 }
)
```

## Steps

### 1. Creating a Docker network
Run the following command to create a network:
```
$ docker network create replicanet
```

### 2. Creating 3 MySQL containers

**One master**
```
docker run -d --rm --name=master --net=replicanet --hostname=master \
  -e MYSQL_ROOT_PASSWORD=mypass \
  --health-cmd='mysqladmin ping -u root -p$${MYSQL_ROOT_PASSWORD}' \
  --health-start-period=10s \
  mysql:5.7.26 \
  --server-id=10 \
  --log-bin='mysql-bin-1.log' \
  --relay_log_info_repository=TABLE \
  --master-info-repository=TABLE \
  --gtid-mode=ON \
  --log-slave-updates=ON \
  --enforce-gtid-consistency
```

**Two slaves**
```
for N in 1 2
do docker run -d --rm --name=slave$N --net=replicanet --hostname=slave$N \
  -e MYSQL_ROOT_PASSWORD=mypass \
  --health-cmd='mysqladmin ping -u root -p$${MYSQL_ROOT_PASSWORD}' \
  --health-start-period=10s \
  mysql:5.7.26 \
  --server-id=$N \
  --relay_log_info_repository=TABLE \
  --master-info-repository=TABLE \
  --gtid-mode=ON \
  --enforce-gtid-consistency \
  --skip-log-slave-updates \
  --skip-log-bin \
  --read_only=TRUE
done
```
Check whether the MySQL container are in `healthy` status before continuing by running `docker ps -a`:
```console
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                    PORTS                 NAMES
658f6f0a0add        mysql:5.7.26        "docker-entrypoint.s…"   40 seconds ago      Up 37 seconds (healthy)   3306/tcp, 33060/tcp   slave2
3d1b84470531        mysql:5.7.26        "docker-entrypoint.s…"   42 seconds ago      Up 40 seconds (healthy)   3306/tcp, 33060/tcp   slave1
3207c898da29        mysql:5.7.26        "docker-entrypoint.s…"   54 seconds ago      Up 52 seconds (healthy)   3306/tcp, 33060/tcp   master
```


### 3. Configuring master slaves replication

Let's configure the **master node**.
```
$ docker exec -it master mysql -uroot -pmypass \
  -e "CREATE USER 'repl'@'%' IDENTIFIED BY 'slavepass';" \
  -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';" \
  -e "SHOW MASTER STATUS;"
```
Master's output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+----------+--------------+------------------+------------------------------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                        |
+--------------------+----------+--------------+------------------+------------------------------------------+
| mysql-bin-1.000003 |      635 |              |                  | 5ae030a1-864c-11e9-810e-0242ac150002:1-7 |
+--------------------+----------+--------------+------------------+------------------------------------------+
```

Let’s configure the **slave nodes** by setting up the master and executing the `START SLAVE;` command.
```
for N in 1 2
  do docker exec -it slave$N mysql -uroot -pmypass \
    -e "CHANGE MASTER TO MASTER_HOST='master', MASTER_USER='repl', \
      MASTER_PASSWORD='slavepass', MASTER_AUTO_POSITION = 1;"

  docker exec -it slave$N mysql -uroot -pmypass -e "START SLAVE;"
done
```

### 4. Checking slaves replication status
* Slave1:
```
$ docker exec -it slave1 mysql -uroot -pmypass -e "SHOW SLAVE STATUS\G"
```
Output:
```console
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin-1.000003
          Read_Master_Log_Pos: 635
               Relay_Log_File: slave1-relay-bin.000003
                Relay_Log_Pos: 852
        Relay_Master_Log_File: mysql-bin-1.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
                             ...

```

* Slave2
```
docker exec -it slave2 mysql -uroot -pmypass -e "SHOW SLAVE STATUS\G"
```
Output:
```console
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin-1.000003
          Read_Master_Log_Pos: 2389
               Relay_Log_File: slave2-relay-bin.000003
                Relay_Log_Pos: 2606
        Relay_Master_Log_File: mysql-bin-1.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
                             ...
```

### 5. Granting access to proxySQL in MySQL and launching ProxySQL container

Run the command below to grant access to `monitor` user:
```
$ docker exec -it master mysql -uroot -pmypass \
  -e "CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor';" \
  -e "GRANT ALL PRIVILEGES on *.* TO 'monitor'@'%';" \
  -e "FLUSH PRIVILEGES;"
```

To run a ProxySQL container with the ProxySQL configuration file:
```
$ docker run -d --rm -p 16032:6032 -p 16033:6033 \
  --name=proxysql --net=replicanet \
  -v $PWD/proxysql.cnf:/etc/proxysql.cnf proxysql/proxysql
```

### 6. Connecting to ProxySQL admin interface

#### Option 1:
```
$ docker exec -it master bash -c 'mysql -hproxysql -P6032 -uradmin -pradmin --prompt "ProxySQL Admin> "'
```
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.5.30 (ProxySQL Admin Module)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

ProxySQL Admin> 
```
The you can execute, for example:
```
ProxySQL Admin> select * from mysql_servers;
```
Output:
```console
+--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 10           | master   | 3306 | 0         | ONLINE | 1      | 0           | 100             | 5                   | 0       | 0              |         |
| 20           | slave1   | 3306 | 0         | ONLINE | 1      | 0           | 100             | 5                   | 0       | 0              |         |
| 20           | slave2   | 3306 | 0         | ONLINE | 1      | 0           | 100             | 5                   | 0       | 0              |         |
+--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
3 rows in set (0.00 sec)
```

#### Option 2:
```
$ docker exec -i master bash -c 'mysql -hproxysql -P6032 -uradmin -pradmin \
   --prompt "ProxySQL Admin> " <<< "select * from mysql_servers;"'
```
Output:
```console
hostgroup_id	hostname	port	gtid_port	status	weight	compression	max_connections	max_replication_lag	use_ssl	max_latency_ms	comment
10	master	3306	0	ONLINE	1	0	100	5	0	0	
20	slave1	3306	0	ONLINE	1	0	100	5	0	0	
20	slave2	3306	0	ONLINE	1	0	100	5	0	0	
```
### 7. Adding data and querying

ProxySQL container doesn't have mysql installed, so we need to access it using one of the running MySQL containers, like:
```
$ docker exec -i master bash -c 'mysql -hproxysql -P6033 -uroot -pmypass -e "SELECT @@port"'
```
Output:
```console
@@port
3306
```
So, let's create a table:
```
$ docker exec -i master bash -c 'mysql -hproxysql -P6033 -uroot -pmypass \
  -e "create database TEST; use TEST; CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY) ENGINE=InnoDB; show tables;"'
```
Adding some data:
```
$ docker exec -i master bash -c 'mysql -hproxysql -P6033 -uroot -pmypass \
  -e "INSERT INTO TEST.t1 VALUES(1); INSERT INTO TEST.t1 VALUES(2); INSERT INTO TEST.t1 VALUES(3);"'
```
Run the select to check the data inserted:
```
$ docker exec -i master bash -c 'mysql -hproxysql -P6033 -uroot -pmypass \
  -e "SELECT * FROM TEST.t1;"'
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
id
1
2
3
```
### 8. Stopping containers, removing created network and image
Stopping running container(s):
```
$ docker stop master slave1 slave2 proxysql
```
Removing the created network:
```
$ docker network rm replicanet
```
Removing MySQL image:
```
$ docker rmi mysql:5.7.26
```
