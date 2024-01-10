---
layout: post
title: MariaDB Replication and its Internals
subtitle: A post about things I learned while playing with MariaDB/MySQL's replication
cover-img: assets/img/cat-glasses-techy.jpg
tags: [databases]
---
Last week, I participated in a hack week organized by [Phil Eaton](https://eatonphil.com) (Thank you, Phil!) with 80 other amazing developers to do fun things with MariaDB/MySQL.  
During the week, I compiled MariaDB on my Mac and looked into the replication code. I set up replication and looked into the code with a debugger to understand how they do it. I will explain my findings in this post; I hope it will be helpful.

# MariaDB/MySQL
[MariaDB](https://mariadb.com) is a fork of the MySQL database. It is open-source ([Github](https://github.com/MariaDB/server)) and developed by interested developers or developers in the MariaDB Foundation.  
## Build and Installation
To build the MariaDB database from source, you can follow the following steps:  
Getting the code:
```bash
git clone https://github.com/MariaDB/server.git mariadb
cd mariadb
git checkout 10.5
```
If you want to have a debug build use `CMake` with the following arguments (If you want to use GDB/LLDB/... you should have a debug build):
```bash
cmake -DCMAKE_BUILD_TYPE=Debug ..
```
If you don't care about debugging you can use the following command:
```bash
cmake -DBUILD_CONFIG=mysql_release .
```
Finally building it:
```bash
make -j4
```
You can install the MariaDB server using:
```
./scripts/mariadb-install-db --srcdir=. --datadir=./datadir
```
You can run the server and the client using the following commands:
```
./sql/mariadbd --defaults-extra-file=./your.cnf
./client/mariadb --defaults-extra-file=./your.cnf
```
# Replication
Replication is the process of copying the data from one place to multiple other places to improve performance, fault tolerance, etc.
MariaDB supports three types of replication:
1. Asynchronous Replication: The master will replicate the data to the followers but doesn't wait for their acknowledgment. 
2. Semisynchronous Replication: The master will wait for at least one replica to commit the transaction.
3. Synchronous Replication: The master will wait for all the replicas to respond. It uses the [Galera Cluster](https://mariadb.com/kb/en/galera-cluster/) to provide synchronous replication. Meta has also implemented [MySQL Raft](https://engineering.fb.com/2023/05/16/data-infrastructure/mysql-raft-meta/) to support replication of MySQL using the Raft consensus algorithm.

I will only discuss the MySQL asynchronous replication in this post.

## How MariaDB/MySQL Performs Async Replication
There are two types of logs that help MariaDB replicate the data from the master database to the slave databases: **binary log** and **relay log**.  
The binary and relay logs have the same format and store information about the transactions so that the slave databases can update their data to be consistent with the master. The master stores the new operations in the binary log and sends the new events to the slaves. Then, the slaves use their relay log to apply the new events to the database.

### Binary Log Format
There are three types of formats to store the logs in the binary log:
1. Statement-based: This format stores the actual SQL statements in the binary log. It is cheap since it doesn't store all the changed information, but might cause inconsistencies if the statement is not deterministic. For example, if the statement has a `RAND()` operation, the slave will have an inconsistent state after applying the SQL statement.
2. Row-based: This format stores the rows changed after applying the SQL statement. It results in larger log files, but it is safer than the statement-based format.
3. Mixed: This format uses a combination of the statement-based and row-based formats. It stores the deterministic operations by storing the statements and stores the non-deterministic operations using the row-based format.

### Replication Internals
The MariaDB code has little documentation for the replication. By running a debugger and exploring the code I have understood the following as the way they do async replication.  
The master waits for the slaves to continuously request new data using the `COM_BINLOG_DUMP` command. The slaves do this in the [`handle_slave_io`](https://github.com/MariaDB/server/blob/d136169e397b594d8285cd7f7d481663417baac6/sql/slave.cc#L4743C3-L4743C3) thread. This thread works like this:
```c++
// Connects to the master (calls the connect_to_master function)
  if (!safe_connect(thd, mysql, mi)) {...}
// Continously read the binary log from the master
  while (!io_slave_killed(mi))
  {
    THD_STAGE_INFO(thd, stage_requesting_binlog_dump);
// Sends the COM_BINLOG_DUMP command to the server to get the binary log
    if (request_dump(thd, mysql, mi, &suppress_warnings))
    { ... }
    ...
    while (!io_slave_killed(mi)) {
        ...
        event_len= read_event(mysql, mi, &suppress_warnings, &network_read_len);
        // Writes the received event to the relay log
        if (queue_event(mi, event_buf, event_len))
        {...}        
    }
    ...
  }
```
The client has another thread called [`handle_slave_sql`](https://github.com/MariaDB/server/blob/d136169e397b594d8285cd7f7d481663417baac6/sql/slave.cc#L5374C5-L5374C5) that applies the log events in the relay log to the database. It works like this: 
```c++
// Opens the relay log
if (init_relay_log_pos(rli,
                        rli->group_relay_log_name,
                        rli->group_relay_log_pos,
                        0 /*need data lock*/, &errmsg,
                        1 /*look for a description_event*/)) {...}
while (!sql_slave_killed(serial_rgi)) {
    ...
    // Execute the next event from the relay log
    if (exec_relay_log_event(thd, rli, serial_rgi))
    {...}
}
```

After getting a `COM_BINLOG_DUMP` command, the master calls [`mysql_binlog_send`](https://github.com/MariaDB/server/blob/d136169e397b594d8285cd7f7d481663417baac6/sql/sql_repl.cc#L2864) from the [`dispatch_command`](https://github.com/MariaDB/server/blob/d136169e397b594d8285cd7f7d481663417baac6/sql/sql_parse.cc#L2122) function to transmit the binary log.

How the master writes the events to the binary log is more complicated. The binary log is declared as a pseudo storage engine:
```c++
maria_declare_plugin(binlog)
{
  MYSQL_STORAGE_ENGINE_PLUGIN,
  &binlog_storage_engine,
  "binlog",
  "MySQL AB",
  "This is a pseudo storage engine to represent the binlog in a transaction",
  PLUGIN_LICENSE_GPL,
  binlog_init, /* Plugin Init */
  NULL, /* Plugin Deinit */
  0x0100 /* 1.0 */,
  binlog_status_vars_top,     /* status variables                */
  binlog_sys_vars,            /* system variables                */
  "1.0",                      /* string version */
  MariaDB_PLUGIN_MATURITY_STABLE /* maturity */
}
maria_declare_plugin_end;
```
The binlog is defined as a handler that writes the transactions or statements to the log.
There are two types of caches used in the binlog handler: `stmt_cache` and `tx_cache`.
If an event is a transaction the handler uses the `tx_cache` to write the event to the binary log upon commit or rollback.
However, if the event is a statement the handler stores it in the `stmt_cache` and waits for the statement to be completed.  
The handler can be found [here](https://github.com/MariaDB/server/blob/d136169e397b594d8285cd7f7d481663417baac6/sql/log.cc#L82).

## Setting Up Async Replication
The server and the slaves should have unique `server_id`s in their configuration for replication to work.  
Example master config file:
```
[mariadb]
log-bin
server_id=1
log-basename=master1
binlog-format=mixed
```
Example slave config file:
```
[mariadb]
server_id=2
```
To set up async replication in MariaDB, you should first make the databases consistent. You can do this using [this](https://mariadb.com/kb/en/setting-up-a-replica-with-mariabackup/) guide.
After that, you should perform the following on the master:
```
CREATE USER 'replication_user'@'localhost' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'localhost';
```
Connect the slave to the master:
```
CHANGE MASTER TO MASTER_HOST='localhost', MASTER_USER='replication_user', MASTER_PASSWORD='password', MASTER_PORT=3307, MASTER_LOG_FILE='master1-bin.000001', MASTER_LOG_POS=689;
START SLAVE;
```
After performing the mentioned steps you should see all the new data on the master to the slave as well.