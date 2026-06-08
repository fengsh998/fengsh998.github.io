---
layout: post
title: "Mqtt Broker for Docker deploy"
categories: k8s
tags: docker
author: fengsh998
typora-root-url: ..
---

Mosquitto 本身没有管理页面，它只是个纯 Broker，
如果需要有管理页面的直接上EMQX

docker-compose.yml
```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"   # MQTT
      - "9001:9001"   # WebSocket（可选，用于网页客户端）
    volumes:
      - ./mosquitto/config/mosquitto.conf:/mosquitto/config/mosquitto.conf:ro
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    environment:
      - TZ=Asia/Shanghai

  # http://<你的服务器IP>:8083
  mqttx-web:
    image: emqx/mqttx-web:latest
    container_name: mqttx-web
    ports:
      - "8083:80" # 将宿主机 8083 端口映射到容器的 80 端口
    restart: unless-stopped
```

mosquitto.conf
```
# mosquitto.conf
# 适用于 STM32 IoT 项目（ESP8266 设备接入）

# ── 监听端口 ──────────────────────────────
listener 1883
protocol mqtt

# WebSocket 支持（网页调试工具用，不需要可删掉）
listener 9001
protocol websockets

# ── 认证 ──────────────────────────────────
# 开发阶段：允许匿名（局域网内可接受）
allow_anonymous true

# 生产阶段：改为以下两行（需先用 mosquitto_passwd 生成密码文件）
# allow_anonymous false
# password_file /mosquitto/config/passwd

# ── 持久化 ────────────────────────────────
persistence true
persistence_location /mosquitto/data/

# ── 日志 ──────────────────────────────────
log_dest file /mosquitto/log/mosquitto.log
log_dest stdout
log_type error
log_type warning
log_type notice
log_type information
# 调试时打开下面这行可以看到每条消息
# log_type all

# ── 连接参数 ──────────────────────────────
max_keepalive 120
# 单客户端最大未确认 QoS1/2 消息数
max_inflight_messages 20

```

创建密码：

```
# 创建密码文件，添加用户 esp_device
docker exec -it mosquitto mosquitto_passwd -c /mosquitto/config/passwd esp_device
```

web页面只用于测试broker是否正常，没办法查看有多少topic正在用

![img](/assets/articles/k8s/emq/emq_connect.jpg)


![img](/assets/articles/k8s/emq/emq-web.jpg)
