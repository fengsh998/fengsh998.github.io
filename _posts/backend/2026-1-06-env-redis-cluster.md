---
layout: post
title: "后端开发快速环境搭建-Redis集群版镜像"
categories: 后端
tags: 开发环境
author: fengsh998
typora-root-url: ..
---

官方不提供集群版的单容器化镜像，因此自制一个。如果只是要单机版的redis 多容器集群的则到官网学习即可。

### 构建Redis单容器集群镜像

我构建时的latest版本为：redis 8.4了

环境：

CentOS 8.0 

Docker V20.10.16

Docker Compose V2.7.0


#### Dockerfile

```Dockerfile
FROM redis:latest

USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends gettext-base bash procps iproute2 \
 && rm -rf /var/lib/apt/lists/*

# 拷贝脚本和模板
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
COPY redis.conf.tmpl /usr/local/etc/redis/redis.conf.tmpl
RUN chmod +x /usr/local/bin/entrypoint.sh

# 暴露 7000-7005 和对应的 bus 17000-17005
EXPOSE 7000-7005 17000-17005

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["foreground"]
```

#### 编写入口文件处理脚本

entrypoint.sh

```shell
#!/bin/bash
set -e

export ANNOUNCE_IP=${ANNOUNCE_IP}
export REDIS_PASSWORD=${REDIS_PASSWORD}
PORTS=(7000 7001 7002 7003 7004 7005)

# 1. 先进行配置准备，但不启动进程
echo ">>> 正在准备节点配置..."
for PORT in "${PORTS[@]}"; do
    export PORT=$PORT
    export BUSPORT=$((PORT + 10000))
    mkdir -p "/data/$PORT"
    envsubst < /usr/local/etc/redis/redis.conf.tmpl > "/data/$PORT/redis.conf"
done

# 2. 【关键修正】在启动进程前，先检查是否需要初始化
# 如果 7000 目录下没有 nodes.conf，说明是真的一点数据都没有
INITIALIZE_CLUSTER=false
if [ ! -f "/data/7000/nodes.conf" ]; then
    INITIALIZE_CLUSTER=true
    echo ">>> 检测到全新环境，标记为待初始化状态..."
fi

# 3. 启动所有 Redis 进程
echo ">>> 启动 Redis 节点进程..."
for PORT in "${PORTS[@]}"; do
    redis-server "/data/$PORT/redis.conf" --daemonize yes
done

sleep 3

# 4. 根据标记执行初始化
if [ "$INITIALIZE_CLUSTER" = true ]; then
    echo ">>> 开始执行集群握手和槽位分配..."
    NODE_IPS=""
    for PORT in "${PORTS[@]}"; do
        NODE_IPS="$NODE_IPS 127.0.0.1:$PORT"
    done
    # 执行初始化
    redis-cli -a "$REDIS_PASSWORD" --cluster create $NODE_IPS --cluster-replicas 1 --cluster-yes
    echo ">>> 集群初始化完成！"
else
    echo ">>> 检测到已有持久化数据，跳过初始化，尝试自动恢复..."
fi

# --- 监控部分 ---
tail -f /data/*/redis.log &

while true; do
    COUNT=$(ps aux | grep redis-server | grep -v grep | wc -l)
    if [ "$COUNT" -lt 6 ]; then
        echo "!!! 节点异常，退出容器..."
        exit 1
    fi
    sleep 5
done
```

#### redis配置文件模板
redis.conf.tmpl
```
# Redis instance config template
port $PORT
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
protected-mode no
bind 0.0.0.0

# announce so clients can connect via mapped host ports / host IP
cluster-announce-ip $ANNOUNCE_IP
cluster-announce-port $PORT
cluster-announce-bus-port $BUSPORT

# password (rendered; if empty the entrypoint will remove these lines)
requirepass $REDIS_PASSWORD
masterauth $REDIS_PASSWORD

# 数据目录
dir /data/$PORT
# 写日志到文件（为了 tail）
logfile /data/$PORT/redis.log
# 可改为 warning 或 debug 看更多日志
loglevel notice

```

文件目录：
        
        -..
          |---- redis-cluster
          |        |------ Dockerfile
          |        |------ entrypoint.sh
          |        |------ redis.conf.tmpl

执行脚本：
```shell
// 进入到目录
cd redis-cluster
//删除旧的镜像
docker rmi my-redis-cluster
//重新构建本地镜像
docker build -t my-redis-cluster:latest .
```

如果没问题的话就成功创建了自己的本地镜像，查看本地镜像docker images
![img](/assets/articles/env/redis-cluster.jpg)

OK ,下面启动一个单容器集群版的redis试下。

docker-compose.yml
```yaml
version: '3.8'

services:
  br-redis-cluster:
    image: my-redis-cluster:latest
    container_name: br-redis-cluster
    ports:
      - "7000-7005:7000-7005"
      - "17000-17005:17000-17005"
    environment:
      - ANNOUNCE_IP=10.49.2.93 //宿主机的IP，必须是 Java 应用能 ping 通的那个 IP
      - REDIS_PASSWORD=myredis!123
    volumes:
      - br_redis_cluster_data:/data

volumes:
  br_redis_cluster_data:

```

执行docker-compose up -d 成功的话可以看到启动单容器集群7000～7005端口，3主3从

docker ps 

![img](/assets/articles/env/redis-cluster1.jpg)


springboot 连接,application.properties

```
spring.redis.timeout = 6000
spring.redis.ssl = false
spring.redis.cluster.masterConnectionPoolSize = 30
spring.redis.cluster.masterConnectionMinimumIdleSize = 5
spring.redis.cluster.slaveConnectionPoolSize = 30
spring.redis.cluster.slaveConnectionMinimumIdleSize = 5
spring.redis.cluster.idleConnectionTimeout = 3600000
spring.redis.cluster.nodes= 10.49.2.93:7000,10.49.2.93:7001,10.49.2.93:7002
spring.redis.password = myredis!123
```







