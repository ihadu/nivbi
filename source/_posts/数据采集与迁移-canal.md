---
title: 数据采集与迁移--canal
date: 2022-03-06 21:00:55
sticky: 1
keywords: 'canal,java'
tags:
 - java
categories: 
 - 大数据组件
 - canal
description: 基于 MySQL 数据库增量日志解析，提供增量数据订阅和消费。可以将mysql数据增量同步至其他存储结构
top_img: 
cover:
password: ihadu1102
---
# 前言
canal，译意为水道/管道/沟渠，
主要用途是基于 MySQL 数据库增量日志解析，提供增量数据订阅和消费。可以将mysql数据增量同步至其他结构,半结构,或非结构数据库例如Redis,Pgsql,Es,kafka,hbase,mangodb等等。

## 工作原理
canal的工作原理就是**把自己伪装成MySQL slave，模拟MySQL slave的交互协议向MySQL Mater发送 dump协议，MySQL mater收到canal发送过来的dump请求，开始推送binary log给canal，然后canal解析binary log，再发送到存储目的地**。

