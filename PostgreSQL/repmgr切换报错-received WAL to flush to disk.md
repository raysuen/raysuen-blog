## 1 报警现象

```
[postgres@ray101 ~]$ repmgr -f /usr/local/pg14/etc/repmgr.conf standby switchover
NOTICE: executing switchover on node "172.16.101" (ID: 1)
NOTICE: attempting to pause repmgrd on 2 nodes
NOTICE: local node "172.16.101" (ID: 1) will be promoted to primary; current primary "172.16.102" (ID: 2) will be demoted to standby
NOTICE: stopping current primary node "172.16.102" (ID: 2)
NOTICE: issuing CHECKPOINT on node "172.16.102" (ID: 2) 
DETAIL: executing server command "/usr/local/pg14/bin/pg_ctl stop -D /pgdata/14/data"
INFO: checking for primary shutdown; 1 of 60 attempts ("shutdown_check_timeout")
NOTICE: current primary has been cleanly shut down at location 0/22000028
NOTICE: waiting up to 30 seconds (parameter "wal_receive_check_timeout") for received WAL to flush to disk
INFO: sleeping 1 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 2 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 3 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 4 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 5 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 6 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 7 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 8 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 9 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 10 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 11 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 12 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 13 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 14 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 15 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 16 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 17 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 18 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 19 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 20 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 21 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 22 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 23 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 24 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 25 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 26 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 27 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 28 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 29 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 30 of maximum 30 seconds waiting for standby to flush received WAL to disk
WARNING: local node "172.16.101" is behind shutdown primary "172.16.102"
DETAIL: local node last receive LSN is 0/21240000, primary shutdown checkpoint LSN is 0/22000028
NOTICE: aborting switchover
HINT: use --always-promote to force promotion of standby
```

切换失败，并且主节点数据库还当关机了，导致业务停摆

## 2 解决方案

```
[postgres@ray102 ~]$ egrep wal_keep_size /pgdata/14/data/postgresql.conf
wal_keep_size=512

[postgres@ray102 ~]$ pg_ctl start
waiting for server to start....2024-05-20 06:29:17.506 GMT [4652] LOG:  00000: redirecting log output to logging collector process
2024-05-20 06:29:17.506 GMT [4652] HINT:  Future log output will appear in directory "log".
2024-05-20 06:29:17.506 GMT [4652] LOCATION:  SysLogger_Start, syslogger.c:674
 done
server started
```

## 3 切换

主数据库修改参数后，切换正常

```
[postgres@ray101 ~]$ repmgr -f /usr/local/pg14/etc/repmgr.conf standby switchover
NOTICE: executing switchover on node "172.16.101" (ID: 1)
NOTICE: attempting to pause repmgrd on 2 nodes
NOTICE: local node "172.16.101" (ID: 1) will be promoted to primary; current primary "172.16.102" (ID: 2) will be demoted to standby
NOTICE: stopping current primary node "172.16.102" (ID: 2)
NOTICE: issuing CHECKPOINT on node "172.16.102" (ID: 2) 
DETAIL: executing server command "/usr/local/pg14/bin/pg_ctl stop -D /pgdata/14/data"
INFO: checking for primary shutdown; 1 of 60 attempts ("shutdown_check_timeout")
NOTICE: current primary has been cleanly shut down at location 0/23000028
NOTICE: promoting standby to primary
DETAIL: promoting server "172.16.101" (ID: 1) using pg_promote()
NOTICE: waiting up to 60 seconds (parameter "promote_check_timeout") for promotion to complete
NOTICE: STANDBY PROMOTE successful
DETAIL: server "172.16.101" (ID: 1) was successfully promoted to primary
WARNING: node "172.16.102" attached in state "startup"
INFO: waiting for node "172.16.102" (ID: 2) to connect to new primary; 1 of max 60 attempts (parameter "node_rejoin_timeout")
DETAIL: node "172.16.101" (ID: 2) is currently attached to its upstream node in state "startup"
NOTICE: node "172.16.101" (ID: 1) promoted to primary, node "172.16.102" (ID: 2) demoted to standby
NOTICE: switchover was successful
DETAIL: node "172.16.101" is now primary and node "172.16.102" is attached as standby
NOTICE: STANDBY SWITCHOVER has completed successfully
```



