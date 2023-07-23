# What is this
My attempt on setting up mysql replica using docker

Resources
- https://www.digitalocean.com/community/tutorials/how-to-set-up-replication-in-mysql
- https://www.linkedin.com/pulse/simplified-guide-mysql-replication-docker-compose-rakesh-shekhawat/


# How to run locally
1. Manually create volumes that will be used in subsequent steps
```bash
$ mkdir master_mysql && touch master_mysql/my.cnf
$ mkdir slave_mysql && touch slave_mysql/my.cnf
```

2. Spin up containers according to [./docker-compose.yml](./docker-compose.yml)
```bash
# spin up
$ docker-compose -f docker-compose.yml up

# verify & expect
$ docker ps

CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS                  PORTS                                                  NAMES
45fb6d406fdf   mysql:latest   "docker-entrypoint.s…"   7 minutes ago   Up Less than a second   33060/tcp, 0.0.0.0:3307->3306/tcp, :::3307->3306/tcp   mysql-slave
862f9268c58e   mysql:latest   "docker-entrypoint.s…"   7 minutes ago   Up Less than a second   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   mysql-master
```

3. Expect [./master_mysql/my.cnf](./master_mysql/my.cnf) and [./slave_mysql/my.cnf](./slave_mysql/my.cnf) to be populated with default values
```cnf
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/8.0/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M

...(rest of file ommited for brevity)
```

4. Update master db's configuration
```diff
# File: master_mysql/my.cnf

skip-name-resolve
datadir=/var/lib/mysql
socket=/var/run/mysqld/mysqld.sock
secure-file-priv=/var/lib/mysql-files
user=mysql

+ bind-address=mysql-master
+ server-id=1
+ log-bin=mysql-bin
+ binlog_do_db=learn_replica_db
```

4. Update slave db's configuration
```diff
# File: slave_mysql/my.cnf

skip-name-resolve
datadir=/var/lib/mysql
socket=/var/run/mysqld/mysqld.sock
secure-file-priv=/var/lib/mysql-files
user=mysql

+ server-id=2
+ log-bin=mysql-bin
+ relay-log=mysql-relay-bin
+ relay-log-index=mysql-relay-bin.index
+ replicate-do-db=learn_replica_db
```

5. Restart containers to take in new config.
```bash
$ docker-compose -f docker-compose.yml down
$ docker-compose -f docker-compose.yml up
```

6. Create replica user in master
```bash
# access into master db docker
docker exec -it mysql-master /bin/bash

# access into mysql using credentials specified in docker-compose.yml for master db
mysql -u root -p

# create replica user
# % means wildcard, i.e. host can be anywhere. Somehow can't get any other options to work, only % works.
CREATE USER 'replica_user'@'%' IDENTIFIED WITH mysql_native_password BY 'replica_user_password';

# verify. expect row to be present
SELECT user, host FROM mysql.user where user = "replica_user";

# grant "REPLICATION SLAVE" permission
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'%';

# verify & expect
SHOW GRANTS FOR 'replica_user'@'%';
+--------------------------------------------------------------+
| Grants for replica_user@%                         |
+--------------------------------------------------------------+
| GRANT REPLICATION SLAVE ON *.* TO `replica_user`@`%` |
+--------------------------------------------------------------+

# best practice, to free up memory in previous operations
FLUSH PRIVILEGES;
```

5. Restart containers to take in new user.
```bash
$ docker-compose -f docker-compose.yml down
$ docker-compose -f docker-compose.yml up
```

7. Retrieving binary log coordinates from the source
```bash
# access into master db docker
docker exec -it mysql-master /bin/bash

# access into mysql using credentials specified in docker-compose.yml for master db
mysql -u root -p

# best practice, read/write lock all tables. Not really necessary here in dev environment, but yes in prod.
FLUSH TABLES WITH READ LOCK;

# verify & expect
SHOW MASTER STATUS;
+------------------+----------+------------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB     | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+------------------+------------------+-------------------+
| mysql-bin.000004 |      157 | learn_replica_db |                  |                   |
+------------------+----------+------------------+------------------+-------------------+

# since we don't have any data to migrate, we'll remove the locks
UNLOCK TABLES;
```

8. Starting and Testing Replication
```bash
# access into slave db docker
docker exec -it mysql-slave /bin/bash

# access into mysql using credentials specified in docker-compose.yml for slave db
mysql -u root -p

# specify config of where to replicate from. These values are referenced from "Create replica user in master" and "Retrieving binary log coordinates from the source" section
CHANGE MASTER TO
MASTER_HOST='mysql-master',
MASTER_USER='replica_user',
MASTER_PASSWORD='replica_user_password',
MASTER_LOG_FILE='mysql-bin.000004',
MASTER_LOG_POS=157;

# start replica server
START REPLICA;

# check status & expect
# the key to look for are
# 1. Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
# 2. Replicate_Do_DB: learn_replica_db
SHOW REPLICA STATUS\G;
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: mysql-master
                  Source_User: replica_user
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-bin.000004
          Read_Source_Log_Pos: 157
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 326
        Relay_Source_Log_File: mysql-bin.000004
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
              Replicate_Do_DB: learn_replica_db
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Source_Log_Pos: 157
              Relay_Log_Space: 536
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Source_SSL_Allowed: No
           Source_SSL_CA_File: 
           Source_SSL_CA_Path: 
              Source_SSL_Cert: 
            Source_SSL_Cipher: 
               Source_SSL_Key: 
        Seconds_Behind_Source: 0
Source_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Source_Server_Id: 1
                  Source_UUID: fa7a404a-28a2-11ee-81f2-0242c0a8f002
             Source_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
    Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Source_Retry_Count: 86400
                  Source_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Source_SSL_Crl: 
           Source_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Source_TLS_Version: 
       Source_public_key_path: 
        Get_Source_public_key: 0
            Network_Namespace: 
1 row in set (0.00 sec)

ERROR: 
No query specified
```

9. Create a table in master db
```bash
# access into master db docker
docker exec -it mysql-master /bin/bash

# access into mysql using credentials specified in docker-compose.yml for master db
mysql -u root -p

# create a dummy tabl
use learn_replica_db
CREATE TABLE example_table (
    example_column varchar(30)
);

# verify and expect
select * from example_table
Empty set (0.00 sec)
```

10. Expect table is replicated in slave
```bash
# access into slave db docker
docker exec -it mysql-slave /bin/bash

# access into mysql using credentials specified in docker-compose.yml for slave db
mysql -u root -p

use learn_replica_db;

# verify and expect
select * from example_table
Empty set (0.00 sec)
```


11. Insert records into master db
```bash
INSERT INTO example_table VALUES
('This is the first row'),
('This is the second row'),
('This is the third row');

# verify and expect
select * from example_table;
+------------------------+
| example_column         |
+------------------------+
| This is the first row  |
| This is the second row |
| This is the third row  |
+------------------------+
3 rows in set (0.00 sec)
```

12. Observe records reflected in slave db
```bash
# verify and expect
select * from example_table;
+------------------------+
| example_column         |
+------------------------+
| This is the first row  |
| This is the second row |
| This is the third row  |
+------------------------+
3 rows in set (0.00 sec)
```

# Hard lessons
1. "Replica failed to initialize applier metadata structure from the repository"
```bash
STOP REPLICA;
RESET REPLICA;
START REPLICA;
SHOW REPLICA STATUS\G;
```

2. The `/etc/mysql/mysql.conf.d/mysqld.cnf` directory in the article is incorret for our context. Since we're using docker, it should be  `/etc/my.conf` [(source)](https://hub.docker.com/_/mysql). Since the docker image doesn't contain any text editor like `vim` or `nano`, the only way is to mount it to our local.
```bash
# manually create directory and file in local
mkdir master_mysql && touch master_mysql/my.cnf

mkdir slave_mysql && touch slave_mysql/my.cnf
```

3. The position of us adding our config into `my.cnf` matters. The values in the article are not suitable, should refer to this https://www.linkedin.com/pulse/simplified-guide-mysql-replication-docker-compose-rakesh-shekhawat/
```cnf

[mysqld]

# needs to be under [mysqld] block
bind-address=mysql-master
server-id=1
log-bin=mysql-bin
binlog_do_db=learn_replica_db

```

4. How to quickly verify if master is ready for replication
```sql
SHOW MASTER STATUS\G;

*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 157
     Binlog_Do_DB: learn_replica_db -- <-- db from my.cnf is passed in correctly
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)

ERROR: 
No query specified
```

5. Tables would be replicated over as well. i.e. if a new table is created in master, it'll be created in slave too.