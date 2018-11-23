## 为什么需要跳板机
1. 尽可能少的将服务器IP暴露在互联网上，从源头断绝端口探测风险；
2. 集群组网的端口开放要求各有不同，尽管一般官方都有端口开放清单，但是运维初次搭建集群，为了搭建工作能顺利进行，一般都会把防火墙关闭，如果服务器IP暴露在外面，那就是裸奔了；
3. 运维人员如果安全意识淡薄，运维上也会留坑：系统弱口令、后台弱口令、服务弱口令等等

## 安装翻墙代理
k8s集群安装和初始化，都是需要翻墙的，出于成本考虑，我们可以把翻墙代理也安装在跳板机上。

> ss服务器密码和谐

- 安装shadowsocks客户端

```bash
# 安装
pip install shadowsocks
# 配置
cat << EOF > /etc/shadowsocks.json
{
    "server":"47.88.57.126",
    "server_port":8989,
    "password":"******",
    "method":"aes-256-cfb"
}
EOF
# 服务
cat << EOF > /etc/systemd/system/shadowsocks.service
[Unit]
Description=Shadowsocks
[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/sslocal -c /etc/shadowsocks.json
[Install]
WantedBy=multi-user.target
EOF
# 启动服务
systemctl enable shadowsocks
service shadowsocks start
# 验证
curl --socks5 127.0.0.1:1080 http://httpbin.org/ip
```

- 安装privoxy

```bash
# 安装
wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -Uvh epel-release-latest-7.noarch.rpm
yum install -y privoxy
# 修改配置 /etc/privoxy/config，默认是监听127.0.0.1，需要修改为内网IP或者0.0.0.0全开放
# forward-socks5t / 127.0.0.1:1080 .
# listen-address  0.0.0.0:8118
# 启动服务
systemctl enable privoxy
service privoxy start
# 验证
curl -x 127.0.0.1:8118 http://httpbin.org/ip
```

> wget, curl 使用代理的方法

```bash
# 全局代理
export http_proxy={proxy_address}
export https_proxy={proxy_address}

# 或者程序代理
wget -e https_proxy={proxy_address} {your_https_url}
wget -e http_proxy={proxy_address} {your_http_url}
curl -x {proxy_address} {your_url}
```