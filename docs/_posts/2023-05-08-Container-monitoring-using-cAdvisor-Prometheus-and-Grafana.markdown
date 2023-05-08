---
layout: post
title: "Container monitoring using cAdvisor, Prometheus, and Grafana"
date: 2023-05-08 09:43:00 +0000
tags: servers
description: "A standalone pipeline for monitoring Docker containers using cAdvisor, Prometheus, and Grafana"
---
# Introduction

`docker-compose.yml` file for monitoring Docker containers using cAdvisor, Prometheus, and Grafana.

```
version: '3.9'
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
    - 9090:9090
    command:
    - --config.file=/etc/prometheus/prometheus.yml
    volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    - type: bind
      source: ./prometheus_data
      target: /prometheus
    depends_on:
    - cadvisor
    restart: always
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
    - 8080:8080
    volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:rw
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
    privileged: true
    restart: always
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
    - 3000:3000
    volumes:
    - type: bind
      source: ./grafana_data
      target: /var/lib/grafana
    depends_on:
    - prometheus
    restart: always
```

`prometheus.yml` file for Prometheus configuration.

```
scrape_configs:
- job_name: cadvisor
  scrape_interval: 5s
  static_configs:
  - targets:
    - cadvisor:8080
```

Use `run_moni.sh` to start the monitoring pipeline.

```
# If there is no grafana_data folder, create it and change permissions to 777
if [ ! -d grafana_data ]; then
    mkdir grafana_data
    chmod -R 777 grafana_data
fi

# If there is no prometheus_data folder, create it and change permissions to 777
if [ ! -d prometheus_data ]; then
    mkdir prometheus_data
    chmod -R 777 prometheus_data
fi

echo killing old docker processes
docker-compose rm -fs

echo building docker containers
docker-compose up --build --remove-orphans -d
```

Now the Grafana dashboard is available at `http://localhost:3000`. Add the prometheus data source at `http://prometheus:9090` and create dashboards of your choice.

Example query for Bytes received per second: `rate(container_network_receive_bytes_total{name=~"container1|container2|container3"}[1m])`, where `container1`, `container2`, and `container3` are the names of the containers to be monitored.

