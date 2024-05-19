# 一 服务器情况：

| 服务器角色 |   服务器IP   |
| :--------: | :----------: |
|  primary   | 172.16.0.101 |
|  standby   | 172.16.0.102 |



# 二安装repmgr

**主服务器已经通过源码安装完成postgresql**

## 2.1 服务器的postgres用户设置ssh互信

```
#node1
ssh-keygen
ssh-copy-id -i /home/postgres/.ssh/id_rsa.pub postgres@node2
```

```
#node2
ssh-keygen
ssh-copy-id -i /home/postgres/.ssh/id_rsa.pub postgres@node1
```

## 2.2 安装repmgr的依赖包(所有节点执行)

```
yum check-update
yum groupinstall -y "Development Tools" 
yum install -y yum-utils openjade docbook-dtds docbook-style-dsssl docbook-style-xsl
yum install -y  yum-builddep flex libselinux-devel libxml2-devel libxslt-devel openssl-devel pam-devel readline-devel
yum install -y libcurl-devel json-c-devel
```

## 2.3 安装repmgr源码包(所有节点执行)

```
[postgres@ray102 ~]$ tar -xzvf repmgr-5.4.1.tar.gz
[postgres@ray102 ~]$ cd repmgr-5.4.1
postgres@ray102 repmgr-5.4.1]$ ./configure 
checking for a sed that does not truncate output... /bin/sed
checking for pg_config... /usr/local/pg14/bin/pg_config
configure: building against PostgreSQL 14.7
checking for gnused... no
checking for gsed... no
checking for sed... yes
configure: creating ./config.status
config.status: creating Makefile
config.status: creating Makefile.global
config.status: creating config.h
[postgres@ray102 repmgr-5.4.1]$ make &&  make install
Building against PostgreSQL 14
gcc -std=gnu99 -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -O2 -fPIC -std=gnu89 -I/usr/local/pg14/include/postgresql/internal -I/usr/local/pg14/include -Wall -Wmissing-prototypes -Wmissing-declarations  -I. -I./ -I/usr/local/pg14/include/postgresql/server -I/usr/local/pg14/include/postgresql/internal  -D_GNU_SOURCE -I/usr/include/libxml2   -c -o repmgr.o repmgr.c
gcc -std=gnu99 -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -O2 -fPIC -shared -o repmgr.so repmgr.o -lcurl -ljson-c -L/usr/local/pg14/lib   -Wl,--as-needed -Wl,-rpath,'/usr/local/pg14/lib',--enable-new-dtags  -L/usr/local/pg14/lib -lpq 
/bin/ld: cannot find -ljson-c
collect2: error: ld returned 1 exit status
make: *** [repmgr.so] Error 1
```

报错解决原因是没有按照json-c-devel，安装后重新make

```
[root@ray101 ~]# rpm -ivh json-c-devel-0.11-4.el7_0.x86_64.rpm 
警告：json-c-devel-0.11-4.el7_0.x86_64.rpm: 头V3 RSA/SHA256 Signature, 密钥 ID f4a80eb5: NOKEY
准备中...                          ################################# [100%]
正在升级/安装...
   1:json-c-devel-0.11-4.el7_0        ################################# [100%]

[root@ray102 repmgr-5.4.1]# make &&  make install
```

## 2.4 主节点数据库创建用户和分配权限(主节点执行)

```
psql -Upostgres -W -d postgres -c "create user repmgr with password '******' superuser replication;"
psql -Upostgres -W -d postgres -c "create database repmgr owner repmgr;"
```

## 2.5 配置访问权限(主节点执行)

```
[postgres@ray101 repmgr-5.4.1]$ vi $PGDATA/pg_hba.conf
local   replication        repmgr                              trust
host    replication        repmgr      127.0.0.1/32            trust
host    replication        repmgr      172.16.0.0/24          trust
local   repmgr        repmgr                              trust
host    repmgr        repmgr      127.0.0.1/32            trust
host    repmgr        repmgr      172.16.0.0/24          trust
```

## 2.6 创建repmgr配置文件(主节点执行)

```
[postgres@ray101 etc]$ pwd
/usr/local/pg14/etc
[postgres@ray101 etc]$ mv repmgr.confi repmgr.conf
[postgres@ray101 etc]$ ls
repmgr.conf
[postgres@ray101 etc]$ cat repmgr.conf 
node_id=1
node_name='172.16.101'
conninfo='host=172.16.101 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/pgdata/14/data'
failover=automatic
promote_command='/usr/local/pg14/bin/repmgr standby promote -f /usr/local/pg14/etc/repmgr.conf --log-to-file'
follow_command='/usr/local/pg14/bin/repmgr standby follow -f /usr/local/pg14/etc/repmgr.conf --log-to-file --upstream-node-id=%n'
service_start_command = '/usr/local/pg14/bin/pg_ctl start -D /pgdata/14/data'
service_stop_command = '/usr/local/pg14/bin/pg_ctl stop -D /pgdata/14/data'
service_restart_command = '/usr/local/pg14/bin/pg_ctl restart -D /pgdata/14/data'
service_reload_command  = '/usr/local/pg14/bin/pg_ctl reload -D /pgdata/14/data'
repmgrd_pid_file='/tmp/repmgrd.pid'
log_file='/usr/local/pg14/etc/repmgrd.log'
priority=100
monitor_interval_secs = 2
connection_check_type ='ping'
reconnect_attempts = 4 
reconnect_interval = 5
use_replication_slots=true
pg_bindir= '/usr/local/pg14/bin'
```

## 2.7 注册主节点(主节点执行)

```
repmgr -f /usr/local/pg14/etc/repmgr.conf primary register
repmgrd -f /usr/local/pg14/etc/repmgr.conf

[postgres@ray101 etc]$ repmgr -f /usr/local/pg14/etc/repmgr.conf primary register
INFO: connecting to primary database...
NOTICE: attempting to install extension "repmgr"
NOTICE: "repmgr" extension successfully installed
NOTICE: primary node record (ID: 1) registered
[postgres@ray101 etc]$ repmgrd -f /usr/local/pg14/etc/repmgr.conf
[2024-05-17 14:49:11] [NOTICE] redirecting logging output to "/usr/local/pg14/etc/repmgrd.log"

repmgr -f /usr/local/pg14/etc/repmgr.conf service status
repmgr -f /usr/local/pg14/etc/repmgr.conf cluster show

[postgres@ray101 etc]$ repmgr -f /usr/local/pg14/etc/repmgr.conf service status

 ID | Name       | Role    | Status    | Upstream | repmgrd     | PID | Paused? | Upstream last seen
----+------------+---------+-----------+----------+-------------+-----+---------+--------------------
 1  | 172.16.101 | primary | * running |          | not running | n/a | n/a     | n/a                
[postgres@ray101 etc]$ repmgr -f /usr/local/pg14/etc/repmgr.conf cluster show
 ID | Name       | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string                                          
----+------------+---------+-----------+----------+----------+----------+----------+-------------------------------------------------------------
 1  | 172.16.101 | primary | * running |          | default  | 100      | 1        | host=172.16.101 user=repmgr dbname=repmgr connect_timeout=2
```

## 2.8 备节点创建repmgr配置文件

```
[postgres@ray102 repmgr-5.4.1]$ mkdir /usr/local/pg14/etc/
[postgres@ray102 repmgr-5.4.1]$ vi /usr/local/pg14/etc/repmgr.conf
[postgres@ray102 repmgr-5.4.1]$ cat /usr/local/pg14/etc/repmgr.conf
node_id=2
node_name='172.16.102'
conninfo='host=172.16.102 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/pgdata/14/data'
failover=automatic
promote_command='/usr/local/pg14/bin/repmgr standby promote -f /usr/local/pg14/etc/repmgr.conf --log-to-file'
follow_command='/usr/local/pg14/bin/repmgr standby follow -f /usr/local/pg14/etc/repmgr.conf --log-to-file --upstream-node-id=%n'
service_start_command = '/usr/local/pg14/bin/pg_ctl start -D /pgdata/14/data'
service_stop_command = '/usr/local/pg14/bin/pg_ctl stop -D /pgdata/14/data'
service_restart_command = '/usr/local/pg14/bin/pg_ctl restart -D /pgdata/14/data'
service_reload_command  = '/usr/local/pg14/bin/pg_ctl reload -D /pgdata/14/data'
repmgrd_pid_file='/tmp/repmgrd.pid'
log_file='/usr/local/pg14/etc/repmgrd.log'
priority=100
monitor_interval_secs = 2
connection_check_type ='ping'
reconnect_attempts = 4 
reconnect_interval = 5
use_replication_slots=true
pg_bindir= '/usr/local/pg14/bin'
```

2.9 克隆主节点数据，并注册备节点到repmgr

```
repmgr -f /usr/local/pg14/etc/repmgr.conf -U repmgr -d repmgr -h 172.16.0.101 standby clone --dry-run -c
repmgr -f /usr/local/pg14/etc/repmgr.conf -U repmgr -d repmgr -h 172.16.0.101 standby clone -c
repmgr -f /usr/local/pg14/etc/repmgr.conf standby register
postgres@ray102 repmgr-5.4.1]$ repmgr -f /usr/local/pg14/etc/repmgr.conf -U repmgr -d repmgr -h 172.16.0.101 standby clone -c
NOTICE: destination directory "/pgdata/14/data" provided
INFO: connecting to source node
DETAIL: connection string is: user=repmgr host=172.16.0.101 dbname=repmgr
DETAIL: current installation size is 50 MB
INFO: replication slot usage not requested;  no replication slot will be set up for this standby
NOTICE: checking for available walsenders on the source node (2 required)
NOTICE: checking replication connections can be made to the source server (2 required)
WARNING: data checksums are not enabled and "wal_log_hints" is "off"
DETAIL: pg_rewind requires "wal_log_hints" to be enabled
INFO: checking and correcting permissions on existing directory "/pgdata/14/data"
NOTICE: starting backup (using pg_basebackup)...
INFO: executing:
  pg_basebackup -l "repmgr base backup"  -D /pgdata/14/data -h 172.16.0.101 -p 5432 -U repmgr -c fast -X stream 
WARNING:  skipping special file "./.s.PGSQL.5432"
WARNING:  skipping special file "./.s.PGSQL.5432"
NOTICE: standby clone (using pg_basebackup) complete
NOTICE: you can now start your PostgreSQL server
HINT: for example: /usr/local/pg14/bin/pg_ctl start -D /pgdata/14/data
HINT: after starting the server, you need to register this standby with "repmgr standby register"
[postgres@ray102 repmgr-5.4.1]$ /usr/local/pg14/bin/pg_ctl start -D /pgdata/14/data
waiting for server to start....2024-05-17 07:10:20.941 GMT [2931] LOG:  00000: redirecting log output to logging collector process
2024-05-17 07:10:20.941 GMT [2931] HINT:  Future log output will appear in directory "log".
2024-05-17 07:10:20.941 GMT [2931] LOCATION:  SysLogger_Start, syslogger.c:674
. done
server started
[postgres@ray102 repmgr-5.4.1]$ repmgr -f /usr/local/pg14/etc/repmgr.conf standby register
INFO: connecting to local node "172.16.102" (ID: 2)
INFO: connecting to primary database
WARNING: --upstream-node-id not supplied, assuming upstream node is primary (node ID: 1)
INFO: standby registration complete
NOTICE: standby node "172.16.102" (ID: 2) successfully registered

[postgres@ray102 repmgr-5.4.1]$ repmgr -f /usr/local/pg14/etc/repmgr.conf service status
 ID | Name       | Role    | Status    | Upstream   | repmgrd     | PID | Paused? | Upstream last seen
----+------------+---------+-----------+------------+-------------+-----+---------+--------------------
 1  | 172.16.101 | primary | * running |            | not running | n/a | n/a     | n/a                
 2  | 172.16.102 | standby |   running | 172.16.101 | not running | n/a | n/a     | n/a                
[postgres@ray102 repmgr-5.4.1]$ repmgr -f /usr/local/pg14/etc/repmgr.conf cluster show
 ID | Name       | Role    | Status    | Upstream   | Location | Priority | Timeline | Connection string                                          
----+------------+---------+-----------+------------+----------+----------+----------+-------------------------------------------------------------
 1  | 172.16.101 | primary | * running |            | default  | 100      | 1        | host=172.16.101 user=repmgr dbname=repmgr connect_timeout=2
 2  | 172.16.102 | standby |   running | 172.16.101 | default  | 100      | 1        | host=172.16.102 user=repmgr dbname=repmgr connect_timeout=2
```

