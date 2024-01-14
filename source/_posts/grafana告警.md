---
abbrlink: ddc4
categories:
  - - tasks
cover: 'https://img.lazysun.me/202307301446894.png'
description: 设置日志，服务器资源告警
tags:
  - deploy
  - prometheus
  - 日志
title: grafana告警
top_img: 'https://img.lazysun.me/202307301447764.jpg'
date: 2023-07-30 12:17:58
updated:aplayer:
aside:
comments:
highlight_shrink:
katex:
keywords:
mathjax:
type:
---

# Grafana告警

继 [服务器监控系统 | lazysun](https://lazysun.me/p/f651.html) ，这次需要设置告警，提前预警可能出现的错误。

## docker安装node-exporter，nprometheus，cadvisor

docker-compose.yaml

```yaml
networks:
  monitoring:
    driver: bridge

services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    expose:
      - 9100
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - '8003:9090'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    expose:
      - 9090
    networks:
      - monitoring

  cadvisor:
    image: google/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    expose:
      - 8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring
```

prometheus.yaml

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090","node-exporter:9100","cadvisor:8080"]
    # 这个是将instance设置为主机名，否则instance会变成"localhost:9090","node-exporter:9100","cadvisor:8080"，很不直观
    relabel_configs:
    - source_labels: [__address__]
      regex: '.*'
      target_label: instance
      replacement: '主机名'
```

之后设置grafana收集prometheus指标即可

## grafana仪表盘

node-exporter推荐: 12385 , 1860

cadvisor推荐：193

loki: 自定义

## grafana设置告警信息

因为我在grafana9.3版本没有找到设置message的字段，所以选择在annotations中加入重要信息解析，然后用webhook发送到转发中间件，再发送到飞书(后面一些原因我将grafana升级到10版本以上后发现有message了)

### 日志告警

使用正则表达式提取日志重要信息

springboot应用日志格式如下:

```java
2023-07-30 12:31:23.826 ERROR 6 --- [127.0.0.1:5672] o.s.a.r.c.CachingConnectionFactory       : Channel shutdown: channel error; protocol method: #method<channel.close>(reply-code=406, reply-text=PRECONDITION_FAILED - delivery acknowledgement on channel 344 timed out. Timeout value used: 1800000 ms. This timeout value can be configured, see consumers doc guide to learn more, class-id=0, method-id=0)
```

正则表达式可如下:

```regexp
(?P<alert_time>\\d{4}-\\d{2}-\\d{2}\\s\\d{1,2}:\\d{2}:\\d{2}.\\d{3})\\s*(?P<level>INFO|ERROR|DEBUG|WARN|CRITICAL)\\s*\\d+\\s*---\\s*\\[(?P<Thread>(.*?))\\]\\s*(?P<logger>(.*?)):\\s*(?P<message>[\\s\\S]*)
```

获得labels: alert_time，level，Thread，logger，message

随后可在grafana alerting中create alert rule，

![202307301351570](https://img.lazysun.me/202307301351570.png)

这里解释一下几个比较重要的参数

告警条件:

```sql
sum by (app, level, alert_time, thread, logger, message)
   (count_over_time(
   {app="user-service", level="ERROR"} 
   != `discard long time none received connection`
   | regexp "(?P<alert_time>\\d{4}-\\d{2}-\\d{2}\\s\\d{1,2}:\\d{2}:\\d{2}.\\d{3})\\s*(?P<level>INFO|ERROR|DEBUG|WARN|CRITICAL)\\s*\\d+\\s*---\\s*\\[(?P<Thread>(.*?))\\]\\s*(?P<logger>(.*?)):\\s*(?P<message>[\\s\\S]*)"[1m]
))
```

每分钟评测一次，当评测sql数据大于0时就会发出告警，告警内容中我在annotaions自定义了header和json属性，header作为飞书卡片的标题，json作为传输数据会被中间件处理，然后分别发出。

json数据:

```json
[{{ range $k, $v := $values }}{"app": "{{ index $v.Labels "app" }}","alertTime": "{{ index $v.Labels "alert_time" }}","level": "<font color='red'>{{ index $v.Labels "level" }}</font>","logger": "{{ index $v.Labels "logger" }}","message": "{{ index $v.Labels "message" }}"},{{ end }}]
```

后面在Notifications设置label，规定发送路由alertRouting，Alert触发后会根据这个参数进行自定义通知。

设置联络点和路由:

![image-20230730140257700](https://img.lazysun.me/202307301402394.png)

### 主机资源告警

设置与日志类似，可以与仪表盘相结合

#### cpu

```sql
1-avg(irate(node_cpu_seconds_total{instance=~"设置的主机名",mode="idle"}[1m]))
```

发送内容(同样由中间件转发):

```
主机: 设置的主机名
cpu使用率大于80,cpu使用率为{{ $values.B0 }}%
```

#### 内存

```sql
(sum(node_memory_MemTotal_bytes{instance="设置的主机名", job="prometheus"} - node_memory_MemAvailable_bytes{instance="设置的主机名", job="prometheus"}) / sum(node_memory_MemTotal_bytes{instance="设置的主机名", job="prometheus"})) *100
```

#### 磁盘空间

```sql
100 - ((node_filesystem_avail_bytes{instance=~"设置的主机名",job=~"prometheus",device!~'rootfs'} * 100) / node_filesystem_size_bytes{instance=~"设置的主机名",job=~"prometheus",device!~'rootfs'})
```

## 总结

网上关于grafana的资料描述的都有些模糊，所以我根据自己的理解自定义了这个告警，使用体验还行，但是因为创建模板使用的模板语言我看不懂，且需要的告警规则不多，我便手动添加了(扩展起来不变),同时使用中间件进行转发[GitHub - zoy0/nacos-notify](https://github.com/zoy0/nacos-notify)，看起来会舒服很多，但是应该是不规范的。