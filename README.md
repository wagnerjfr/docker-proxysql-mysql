# Using Docker containers to setup ProxySQL + MySQL Replication

**ProxySQL** (https://proxysql.com/) is an open-source MySQL proxy server, meaning it serves as an intermediary between a MySQL server and the applications that access its databases. ProxySQL can improve performance by distributing traffic among a pool of multiple database servers and also improve availability by automatically failing over to a standby if one or more of the database servers fail.

## ProxySQL config file

Let's see the main definitions we have in `proxysql.cnf` config file

Defines the proxySQL admin credentials and port to access `6032`.
```
admin_variables=
{
    admin_credentials="admin:admin;radmin:radmin"
    mysql_ifaces="0.0.0.0:6032"
    refresh_interval=2000
}
```

Defines the hostgroups and the role of which MySQL server.
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

MySQL user:
```
mysql_users =
(
    { username = "root" , password = "mypass" , default_hostgroup = 10 , active = 1 }
)
```

## Steps

### 1. Creating a Docker network
Fire the following command to create a network:
```
docker network create replicanet
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

### 3. Configuring replication amonng master and slaves

Let's configure the **master node**.
```
docker exec -it master mysql -uroot -pmypass \
  -e "CREATE USER 'repl'@'%' IDENTIFIED BY 'slavepass';" \
  -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';" \
  -e "SHOW MASTER STATUS;"
```

Let’s continue with the **slave nodes**.
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
docker exec -it slave1 mysql -uroot -pmypass -e "SHOW SLAVE STATUS\G"
```
Slave1 output:
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
Slave2 output:
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

### 5. Granting access to proxySQL in MySQL and launching ProxySQL

Run the command below to grant access to `monitor` user:
```
docker exec -it master mysql -uroot -pmypass \
-e "CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor';" \
-e "GRANT ALL PRIVILEGES on *.* TO 'monitor'@'%';" \
-e "FLUSH PRIVILEGES;"
```

Execute the command to laucnh the proxySQL container:
```
docker run -d --rm -p 16032:6032 -p 16033:6033 \
  --name=proxysql --net=replicanet \
  -v $PWD/proxysql.cnf:/etc/proxysql.cnf proxysql/proxysql
```
