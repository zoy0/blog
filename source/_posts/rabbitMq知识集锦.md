---
abbrlink: '5180'
categories:
  - - note
cover: 'https://img.lazysun.me/cover-202310092119179.webp'
description: 设置日志，服务器资源告警
tags:
  - MQ
title: RabbitMq知识集锦
top_img: 'https://img.lazysun.me/202307301447764.jpg'
date: 2023-09-14 12:50:22
updated:aplayer:
aside:
comments:
highlight_shrink:
katex:
keywords:
mathjax:
type:
---





# RabbitMq知识集锦

## rabbitmq基本概念

略

## 相关问题

### 1. 为什么推荐用MQ取代线程池？

多线程的好处是增加服务器并行处理的效率，一般用于大量数据处理比较耗时，将其分为多个块，并行处理；或者外部接口调用比较耗时，不想堵塞同步接口。并且线程有个丢弃策略，消息丢失是否可以接受也是有待商议的。所以有一个方法就是通过mq将手里的活交给其他服务去处理。

MQ的好处:

1. 为了降低rt，有时候我们会使用mq进行自发自收
2. 临时数据存储，能处理多少数据不再依赖于jvm的内存大小，而取决于mq的配置
3. 处理能力的可控性，通过查看mq管控平台得知消费和生产速率，决定是否要扩缩容
4. 可靠性，提供可持久化，宕机或者重启可恢复数据，而线程池的数据存在于内存，重启就没了
5. 可扩展性好，不会因为单机瓶颈影响性能
6. 跨语言平台，可用其他高性能语言作为消费者处理消息，加快消费速率

看 [为什么我推荐用MQ取代线程池？_哔哩哔哩_bilibili ](https://www.bilibili.com/video/BV1Aw411i7jx/)  总结

