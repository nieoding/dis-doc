## 1. Hello World

- index.php

```
<?php
echo '<h1>Hello World!</h1>'
?>
```
- Dockerfile

```bash
from centos:7
run yum install -y php
add index.php /
cmd "php" "-S" "0.0.0.0:8000"
```
- 构建
 
```bash
[root@localhost ~]#  docker build -t mycontainer .
Sending build context to Docker daemon  3.072kB
Step 1/4 : from centos:7
 ---> 5182e96772bf
Step 2/4 : run yum install -y php
 ---> Using cache
 ---> 6a29566350ac
Step 3/4 : add index.php /
 ---> Using cache
 ---> fbac482cee68
Step 4/4 : cmd "php" "-S" "0.0.0.0:8000"
 ---> Using cache
 ---> 3eb58ce8991e
Successfully built 3eb58ce8991e
Successfully tagged mycontainer:latest
```
- 运行

```bash
[root@localhost ~]# docker run -it -p 8001:8000 mycontainer
PHP 5.4.16 Development Server started at Sun Nov 18 04:15:54 2018
Listening on http://0.0.0.0:8000
Document root is /
Press Ctrl-C to quit.
```
- 发布

> 1. https://hub.docker.com注册
> 
> 2. 命令行登录`docker login`
> 
> 3. 给镜像打标签`docker tag mycontainer dingzhihong/mytest`
> 
> 4. 提交`docker push dingzhihong/mytest`

- tips

```bash
# 容器后台运行
docker run -p 8001:8000 -d mycontainer 
# 一直监控容器日志
docker logs -f {containerid}
# 进入容器
docker exec -it {containerid} bash
```

## 2. 提问
1. cmd 和 entrypoint 的异同
2. 编译镜像如何瘦身
3. 解读并编译redis镜像

```bash
FROM debian:stretch-slim

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r redis && useradd -r -g redis redis

# grab gosu for easy step-down from root
# https://github.com/tianon/gosu/releases
ENV GOSU_VERSION 1.10
RUN set -ex; \
	\
	fetchDeps=" \
		ca-certificates \
		dirmngr \
		gnupg \
		wget \
	"; \
	apt-get update; \
	apt-get install -y --no-install-recommends $fetchDeps; \
	rm -rf /var/lib/apt/lists/*; \
	\
	dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \
	wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch"; \
	wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc"; \
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
	gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
	gpgconf --kill all; \
	rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc; \
	chmod +x /usr/local/bin/gosu; \
	gosu nobody true; \
	\
	apt-get purge -y --auto-remove $fetchDeps

ENV REDIS_VERSION 5.0.1
ENV REDIS_DOWNLOAD_URL http://download.redis.io/releases/redis-5.0.1.tar.gz
ENV REDIS_DOWNLOAD_SHA 82a67c0eec97f9ad379384c30ec391b269e17a3e4596393c808f02db7595abcb

# for redis-sentinel see: http://redis.io/topics/sentinel
RUN set -ex; \
	\
	buildDeps=' \
		ca-certificates \
		wget \
		\
		gcc \
		libc6-dev \
		make \
	'; \
	apt-get update; \
	apt-get install -y $buildDeps --no-install-recommends; \
	rm -rf /var/lib/apt/lists/*; \
	\
	wget -O redis.tar.gz "$REDIS_DOWNLOAD_URL"; \
	echo "$REDIS_DOWNLOAD_SHA *redis.tar.gz" | sha256sum -c -; \
	mkdir -p /usr/src/redis; \
	tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1; \
	rm redis.tar.gz; \
	\
# disable Redis protected mode [1] as it is unnecessary in context of Docker
# (ports are not automatically exposed when running inside Docker, but rather explicitly by specifying -p / -P)
# [1]: https://github.com/antirez/redis/commit/edd4d555df57dc84265fdfb4ef59a4678832f6da
	grep -q '^#define CONFIG_DEFAULT_PROTECTED_MODE 1$' /usr/src/redis/src/server.h; \
	sed -ri 's!^(#define CONFIG_DEFAULT_PROTECTED_MODE) 1$!\1 0!' /usr/src/redis/src/server.h; \
	grep -q '^#define CONFIG_DEFAULT_PROTECTED_MODE 0$' /usr/src/redis/src/server.h; \
# for future reference, we modify this directly in the source instead of just supplying a default configuration flag because apparently "if you specify any argument to redis-server, [it assumes] you are going to specify everything"
# see also https://github.com/docker-library/redis/issues/4#issuecomment-50780840
# (more exactly, this makes sure the default behavior of "save on SIGTERM" stays functional by default)
	\
	make -C /usr/src/redis -j "$(nproc)"; \
	make -C /usr/src/redis install; \
	\
	rm -r /usr/src/redis; \
	\
	apt-get purge -y --auto-remove $buildDeps

RUN mkdir /data && chown redis:redis /data
VOLUME /data
WORKDIR /data

COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 6379
CMD ["redis-server"]
```
