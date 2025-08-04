---
title: Grafana+Prometheus监控
published: 2024-12-21
tags:
  - OS
  - Software
category: Zheteng
draft: false
---

记一下Prometheus+Grafana部署

```yaml
services:
  grafana:
    image: grafana/grafana-enterprise
    container_name: grafana
    restart: unless-stopped
    environment:
     - GF_SERVER_ROOT_URL=https://grafana.sunnypai.top/
    networks:
      - caddy_network
      - grafana_network
#    ports:
#     - '3000:3000'
    volumes:
     - ./grafana:/var/lib/grafana

  pushgateway:
    image: prom/pushgateway:v1.10.0
    container_name: pushgateway
    command: --persistence.file=/pushgateway/pushgateway.data
    restart: unless-stopped
    networks:
      - grafana_network
#    ports:
#      - 9091:9091
    volumes:
      - ./pushgateway:/pushgateway

  prometheus:
    image: prom/prometheus:v3.0.1
    container_name: prometheus
    command: --config.file=/etc/prometheus/prometheus.yml
    user: 0:0
    restart: unless-stopped
    networks:
      - grafana_network
#    ports:
#      - 9090:9090
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alerts/:/etc/prometheus/rules.d/
      - ./prometheus/data/:/prometheus

  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: alertmanager
    command: --config.file=/etc/alertmanager/alertmanager.yml
    networks:
      - grafana_network
#    ports:
#      - 9093:9093
    restart: unless-stopped
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml

  node-exporter:
    image: "prom/node-exporter:v1.3.1"
    container_name: node-exporter
#    ports:
#      - '9100:9100'
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
      - /etc/hostname:/etc/hostname:ro
    restart: unless-stopped
    networks:
      - grafana_network
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.45.0
    container_name: cadvisor
#    ports:
#      - "8080:8080"
    networks:
      - grafana_network
    volumes:
      - "/:/rootfs:ro"
      - "/var/run:/var/run:ro"
      - "/sys:/sys:ro"
      - "/var/lib/docker/:/var/lib/docker:ro"
      - "/dev/disk/:/dev/disk:ro"
    privileged: true
    devices:
      - "/dev/kmsg"
    restart: unless-stopped

  blackbox-exporter:
    image: prom/blackbox-exporter:v0.25.0
    container_name: blackbox-exporter
#    ports:
#      - "9115:9115"
    networks:
      - grafana_network
    volumes:
      - "./blackbox:/config"
    command:
      - "--config.file=/config/blackbox.yml"
    restart: unless-stopped

networks:
  caddy_network:
    external: true
  grafana_network:
    external: true
```

容器的作用：

- Grafana: 图形化WebUI
- Prometheus: 数据源聚合
- pushgateway: 边缘节点数据采集
- alertmanager: 告警通知
- node-exporter: 节点探针
- cAdvisor: 容器状态监控
- blackbox-exporter: Web服务状态监控

边缘节点相关配置未完待续……

2025年8月4日更新：Grafana+Prometheus对内存占用太恐怖了，拆了跑路了，没有后续了