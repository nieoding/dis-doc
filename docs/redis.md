![](img/redis.jpg)
> redis从5开始已经废弃了使用ruby脚本创建集群，而将创建集群指令直接集成在了redis-cli里面

生产环境建议采用 3master + 3slave 总共6台独立服务器搭建

## 安装
- 安装版本v5

```bash
yum install -y http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
yum --enablerepo=remi install -y redis
systemctl enable redis
```
- 修改配置，切换到集群模式

> 必须绑定私有IP，而不是0.0.0.0 (估计5有BUG，无法正常寻址)

```bash
# /etc/redis.conf
bind {private_ip}
cluster-enabled yes
```
- 启动

```
service redis start
```

## 创建集群
任意节点下执行

```bash
redis1=192.168.0.166
redis2=192.168.0.169
redis3=192.168.0.168
redis1_slave=192.168.0.164
redis2_slave=192.168.0.167
redis3_slave=192.168.0.165

redis-cli --cluster create ${redis1}:6379 ${redis2}:6379 ${redis3}:6379 ${redis1_slave}:6379 ${redis2_slave}:6379 ${redis3_slave}:6379 --cluster-replicas 1
```

正常的话，屏幕输出如下

```bash
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.0.164:6379 to 192.168.0.166:6379
Adding replica 192.168.0.167:6379 to 192.168.0.169:6379
Adding replica 192.168.0.165:6379 to 192.168.0.168:6379
M: c16eae487165b689c195c88feac45e2cd1eb979b 192.168.0.166:6379
   slots:[0-5460] (5461 slots) master
M: a4f326cf24260dbb3f669e9eefa38a2fafc27409 192.168.0.169:6379
   slots:[5461-10922] (5462 slots) master
M: 80eb826d4db28be85c7795cfd40b3268bcf2296a 192.168.0.168:6379
   slots:[10923-16383] (5461 slots) master
S: e1a1b24711a05f08e15269eb9a508852301d1f45 192.168.0.164:6379
   replicates c16eae487165b689c195c88feac45e2cd1eb979b
S: 465e50a0196dbb641e44a733547328b6c4d0adfe 192.168.0.167:6379
   replicates a4f326cf24260dbb3f669e9eefa38a2fafc27409
S: 8e7febc522a3d07a9d56d4e5ce88895b530a5f9f 192.168.0.165:6379
   replicates 80eb826d4db28be85c7795cfd40b3268bcf2296a
Can I set the above configuration? (type 'yes' to accept):
```

确认没问题，输入yes继续

```bash
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
```

集群内部开始相互握手，过10秒钟握手完毕，系统开始初始化，分片

```bash
>>> Performing Cluster Check (using node 192.168.0.166:6379)
M: c16eae487165b689c195c88feac45e2cd1eb979b 192.168.0.166:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 465e50a0196dbb641e44a733547328b6c4d0adfe 192.168.0.167:6379
   slots: (0 slots) slave
   replicates a4f326cf24260dbb3f669e9eefa38a2fafc27409
S: 8e7febc522a3d07a9d56d4e5ce88895b530a5f9f 192.168.0.165:6379
   slots: (0 slots) slave
   replicates 80eb826d4db28be85c7795cfd40b3268bcf2296a
M: a4f326cf24260dbb3f669e9eefa38a2fafc27409 192.168.0.169:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: e1a1b24711a05f08e15269eb9a508852301d1f45 192.168.0.164:6379
   slots: (0 slots) slave
   replicates c16eae487165b689c195c88feac45e2cd1eb979b
M: 80eb826d4db28be85c7795cfd40b3268bcf2296a 192.168.0.168:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

## 安装监控

> 可以单独搭建在独立服务器上

- 安装脚本

```
yum install -y ruby ruby-devel gcc gcc-c++
gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
gem install redis-stat

```
- 启动脚本

```
redis-stat redis1 redis2 redis3 redis1_slave redis2_slave redis3_slave  --server --daemon
```
> 建议将6台服务器地址写入/etc/hosts

> 参数 --server --daemon 是后台模式运行，默认监听63790端口

- 指标参数清单

简写|	指标|	说明
---|---|---
us|	used_cpu_user|	用户空间占用CPU百分比
sy|	used_cpu_sys|	内核空间占用CPU百分比
cl|	connected_clients|	连接客户端数量
bcl|	blocked_clients|	阻塞客户端数量(如BLPOP)
mem|	used_memory|	使用总内存
rss|	used_memory_rss|	使用物理内存
keys|	dbx.keys|	key的总数量
cmd/s|	command/s|	每秒执行命令数
exp/s|	expired_keys/s|	每秒过期key数量
evt/s|	evicted_keys/s|	每秒淘汰key数量
hit%/s|	keyspace_hitratio/s|	每秒命中百分比
hit/s|	keyspace_hits/s|	每秒命中数量
mis/s|	keyspace_miss/s|	每秒丢失数量
aofcs|	aof_current_size|	AOF日志当前大小

## LB

```bash
# /etc/haproxy/haproxy.cfg

listen  redis *:6379
    mode tcp
    balance     roundrobin
    timeout client  3h
    timeout server  3h
    option          clitcpka
    server _1 redis1:6379 check inter 5s rise 2 fall 3
    server _2 redis2:6379 check inter 5s rise 2 fall 3
    server _3 redis3:6379 check inter 5s rise 2 fall 3
    server _4 redis1-slave:6379 check inter 5s rise 2 fall 3
    server _5 redis2-slave:6379 check inter 5s rise 2 fall 3
    server _6 redis3-slave:6379 check inter 5s rise 2 fall 3
```

## 性能分析方法

- 压测方法

```
redis-benchmark -t set,get -n 100000 -h {addr}
```
> 8w/s为good

- 延时分析

```
redis-cli --latency -h {addr}
```
> avg稳定在0.3-0.4为good

- 慢查询分析

```
:6379> slowlog get 
```
