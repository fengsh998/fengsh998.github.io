---
layout: post
title: "后端开发快速环境搭建-kafka,mq,redis,mongodb"
categories: 后端
tags: 开发环境
author: fengsh998
typora-root-url: ..
---

环境：

CentOS 8.0 

Docker V20.10.16

Docker Compose V2.7.0


### 构建一条龙docker-compose

```yaml
version: '3.8'

# 明确定义自定义网络，所有服务默认加入
networks:
  br-app-network:
    name: br-app-network
    driver: bridge

services:
  br-redis-cluster:
    image: my-redis-cluster:latest
    container_name: br-redis-cluster
    ports:
      - "7000-7005:7000-7005"
      - "17000-17005:17000-17005"
    environment:
      - ANNOUNCE_IP=10.x.x.x  #改成自己的宿主机IP
      - REDIS_PASSWORD=myredis!123
    volumes:
      - br_redis_cluster_data:/data
    networks:
      - br-app-network

  rabbitmq:
    image: rabbitmq:3-management #官方没带插件
    container_name: br-rabbitmq
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # 管理界面 http://localhost:15672 (guest/guest)
    volumes:
      - br_rabbitmq_data:/var/lib/rabbitmq
      - br_rabbitmq_log:/var/log/rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    networks:
      - br-app-network

  kafka:
    image: apache/kafka:latest  # 官方 KRaft 模式
    container_name: br-kafka
    ports:
      - "9092:9092"  # 宿主机访问也可用 localhost:9092
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      # 关键：其他容器通过 kafka:9092 连接 PLAINTEXT://kafka:9092
      # 重点：如果是宿主机应用访问，这里必须包含宿主机 IP (10.49.4.97) 或 localhost
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://10.49.4.97:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      CLUSTER_ID: Mk3OEYBSD34fcwNTJENDM2Qk  # 固定集群 ID，可自行生成
    volumes:
      - br_kafka_data:/var/lib/kafka/data
    networks:
      - br-app-network


  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: br-kafka-ui
    restart: unless-stopped
    ports:
      - "21944:8080" 
    environment:
      # 定义集群名称（页面上显示的标题）
      KAFKA_CLUSTERS_0_NAME: local-cluster
      # 关键：连接到你上面定义的 kafka 服务名
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      # 如果需要支持在界面上创建/删除 Topic，可以加上这个权限
      KAFKA_CLUSTERS_0_READONLY: "false"
      # 设置默认的动态加载（可选）
      DYNAMIC_CONFIG_ENABLED: 'true'
    depends_on:
      - kafka
    networks:
      - br-app-network

  mysql:
    image: mysql:8.0  # 官方最新稳定版（也可指定 mysql:latest）
    container_name: br-mysql
    restart: unless-stopped
    ports:
      - "3306:3306"  # 宿主机访问
    environment:
      MYSQL_ROOT_PASSWORD: mysql!123     # root 用户密码（请修改为安全密码）
      #MYSQL_DATABASE: appdb                 # 默认创建数据库
      #MYSQL_USER: appuser                   # 创建普通用户
      #MYSQL_PASSWORD: apppassword           # 普通用户密码（请修改）
    volumes:
      - br_mysql_data:/var/lib/mysql           # 数据持久化
      # 如需初始化脚本，可挂载目录：
      # - ./mysql-init:/docker-entrypoint-initdb.d/
    command: --default-authentication-plugin=mysql_native_password  # 兼容旧客户端
    networks:
      - br-app-network

  mongodb:
    image: mongo:8.0.0  # 官方最新版8.2.3 
    container_name: br-mongodb
    restart: unless-stopped
    ports:
      - "0.0.0.0:27017:27017"  # MongoDB 默认端口
    environment:
      MONGO_INITDB_ROOT_USERNAME: root       # root 用户
      MONGO_INITDB_ROOT_PASSWORD: rootpassword  # root 密码（请修改）
      #MONGO_INITDB_DATABASE: appdb           # 初始化数据库（可选）
    volumes:
      - br_mongodb_data:/data/db                # 数据持久化
      # 如需初始化脚本，可挂载：
      # - ./mongo-init:/docker-entrypoint-initdb.d/
    networks:
      - br-app-network

  # 可选：Mongo Express（Web 管理界面，类似 phpMyAdmin）
  mongo-express:
    image: mongo-express:latest
    container_name: br-mongo-express
    restart: unless-stopped
    ports:
      - "8081:8081"  # 访问 http://localhost:8081
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: root
      ME_CONFIG_MONGODB_ADMINPASSWORD: rootpassword
      ME_CONFIG_MONGODB_URL: mongodb://root:rootpassword@mongodb:27017/
      ME_CONFIG_BASICAUTH: false  # 开发时关闭基本认证，生产请开启
    depends_on:
      - mongodb
    networks:
      - br-app-network

volumes:
  br_redis_cluster_data:
  br_rabbitmq_data:
  br_rabbitmq_log:
  br_kafka_data:
  br_mysql_data:
  br_mongodb_data:

```

基中redis使用了单容器集群版的镜像。

### 使用过程中碰到问题

```
[TID: N/A] [main] INFO  org.mongodb.driver.cluster:71 - Cluster created with settings {hosts=[10.x.x.x:27017], mode=SINGLE, requiredClusterType=UNKNOWN, serverSelectionTimeout='30000 ms', maxWaitQueueSize=500}
2025-12-26 09:25:09.950 +0800 - [TID: N/A] [cluster-ClusterId{value='694de3f5cd31643e3b929bf2', description='null'}-10.x.x.x:27017] INFO  org.mongodb.driver.cluster:76 - Exception in monitor thread while connecting to server 10.x.x.x:27017
com.mongodb.MongoCommandException: Command failed with error 13 (Unauthorized): 'Command buildInfo requires authentication' on server 10.x.x.x:27017. The full response is { "ok" : 0.0, "errmsg" : "Command buildInfo requires authentication", "code" : 13, "codeName" : "Unauthorized" }
```

一直报连接授权出错，原来我docker装的mongodb是8.2.3版本，使用java spring死活连不上，后来我降到了mongodb 8.0.0就正常了

        //列出 volume
        docker volume ls
        //删除所有持久化
        docker compose down --volumes 或 -v 
        删除一个
        docker volume rm yourproject_mongodb_data
        删除多个
        docker volume rm yourproject_mongodb_data yourproject_mysql_data
