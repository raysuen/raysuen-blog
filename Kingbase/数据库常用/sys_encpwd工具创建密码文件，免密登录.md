在很多时候需要免密登录，又不能改变安全访问策略，所以使用密码文件就是一个好的方案，通过加密后的密码文件免密访问数据库。



### 命令的帮助

```
[kingbase@ray22 bin]$ ./sys_encpwd --help

[-H, --hostname=]         host name
[-P, --portnum=]          port number
[-D, --database=]         database name
[-U, --user=]           user name
[-W, --password=]         password
```



### 在密码文件内生产加密策略

```
[kingbase@ray22 bin]$ cat ~/.encpwd
*:*:*:esrep:S2luZ2Jhc2VoYTExMA==
```

```
[kingbase@ray22 bin]$ ./sys_encpwd -H \* -P \* -D \* -U system -W vbox8269#
```

```
[kingbase@ray22 bin]$ cat ~/.encpwd
*:*:*:esrep:S2luZ2Jhc2VoYTExMA==
*:*:*:system:dmJveDgyNjkj
```



### 验证

```
[kingbase@ray22 bin]$ egrep "^local" ../data/sys_hba.conf
local  all       all                   scram-sha-256
local  replication   all                   scram-sha-256

[kingbase@ray22 bin]$ ./ksql test system
输入 "help" 来获取帮助信息.

test=#
```

