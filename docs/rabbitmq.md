> 集群模式

![](img/mq_cluster.jpg)

> 路由模式

![](img/mq_router.jpg)

## 配置代理
> 假设3台服务器都无法访问外网，使用跳板机做访问代理（如果可以访问外网，可以跳过该步骤）

```
proxy={proxy_address}

export https_proxy=$proxy
export http_proxy=$proxy
```


## 安装 & 配置环境
> 服务器分别命名mq1, mq2, mq3

服务器 | ip
---|---
mq1|192.168.0.111
mq2|192.168.0.112
mq3|192.168.0.113

下面很多地方会用到hostname来标识节点名称，所以在每台服务器上配置一下hosts

```bash
mq1=192.168.0.111
mq2=192.168.0.112
mq3=192.168.0.113

cat << EOF >> /etc/hosts
$mq1    mq1
$mq2    mq2
$mq3    mq3
EOF
```

在每台服务器上安装好rabbitmq-server服务

```bash
cat << EOF > /etc/yum.repos.d/rabbitmq-erlang.repo
[rabbitmq-erlang]
name=rabbitmq-erlang
baseurl=https://dl.bintray.com/rabbitmq/rpm/erlang/21/el/7
gpgcheck=1
gpgkey=https://dl.bintray.com/rabbitmq/Keys/rabbitmq-release-signing-key.asc
repo_gpgcheck=0
enabled=1
EOF
yum install -y socat erlang

wget https://dl.bintray.com/rabbitmq/all/rabbitmq-server/3.7.7/rabbitmq-server-3.7.7-1.el7.noarch.rpm -O mq.rpm
rpm -Uvh mq.rpm && rm -f mq.rpm

mkdir /etc/systemd/system/rabbitmq-server.service.d/
cat << EOF > /etc/systemd/system/rabbitmq-server.service.d/limits.conf
[Service]
LimitNOFILE=300000
EOF

systemctl enable rabbitmq-server
service rabbitmq-server start
rabbitmq-plugins enable rabbitmq_management
```

## 集群组建
- 同步cookie

> mq1上执行

```bash
scp /var/lib/rabbitmq/.erlang.cookie mq2:
scp /var/lib/rabbitmq/.erlang.cookie mq3:
```

> mq2, mq3上执行

```
chown rabbitmq:rabbitmq .erlang.cookie
chmod 400 .erlang.cookie
mv -f .erlang.cookie /var/lib/rabbitmq/
```

- 加入集群

> 在mq2, mq3上执行

```
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@mq1
rabbitmqctl start_app

# 验证
rabbitmqctl cluster_status
```
> rabbitmq的日志存放在`/var/logs/rabbitmq`目录里，如果遇到意外错误，查看日志排查原因

- 设置策略

> 任意节点执行如下代码则切换到集群模式（如果采用系统默认的路由模式，可以跳过此步）

```
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```

## 创建用户

> 任意节点执行

```bash
username=dingding
password=dingding1234

# 添加用户
rabbitmqctl add_user ${username} ${password}

# 加入管理员
rabbitmqctl set_user_tags ${username} administrator

# 对根目录设置消费权限
rabbitmqctl set_permissions -p / ${username} ".*" ".*" ".*"

```


## LB
建议将mq1, mq2, mq3写在 /etc/hosts里面

```bash
# /etc/haproxy/haproxy.cfg
listen  mq *:5672
    mode tcp
    balance     roundrobin
    timeout client  3h
    timeout server  3h
    option          clitcpka
    server _1 mq1:5672 check inter 5s rise 2 fall 3
    server _2 mq2:5672 check inter 5s rise 2 fall 3
    server _3 mq3:5672 check inter 5s rise 2 fall 3
listen mq-web *:15672
    mode http
    balance     roundrobin
    server _1 mq1:15672
    server _2 mq2:15672
    server _3 mq3:15672
```