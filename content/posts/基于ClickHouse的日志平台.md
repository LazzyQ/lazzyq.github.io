---
title: "基于ClickHouse的日志平台"
date: 2020-12-03T21:07:02+08:00
draft: true
original: true
categories: 
  - 大数据
tags: 
  - ClickHouse
---

filebeat.yml

```
# https://github.com/elastic/beats/blob/master/filebeat/filebeat.reference.yml

filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/*.log
    json.keys_under_root: true
output.kafka:
  enabled: true
  hosts: ["sx-kafka1:9092", "sx-kafka2:9092","sx-kafka3:9092"]
  required_acks: 1
```




```
create database kafka_test on cluster cluster_3s_2r

use  kafka_test

CREATE TABLE kafka_msg (
  trace String, 
  app String,
  file String,
  func String,
  level String,
  msg String,
  body String,
  custom String,
  ts DateTime
) ENGINE = Kafka SETTINGS kafka_broker_list = '10.0.2.0:9092,10.0.2.1:9092,10.0.2.2:9092',
                        kafka_topic_list = 'beats',
                        kafka_group_name = 'clickhouse_group',
                        kafka_format = 'JSONEachRow';

CREATE TABLE kafka_msg_daily (
  trace String,
  app String,
  file String,
  func String,
  level String,
  msg String,
  ts DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts) ORDER BY (trace, ts);

CREATE MATERIALIZED VIEW kafka_msg_daily_view TO kafka_msg_daily AS SELECT trace,file,func,level,msg,ts FROM kafka_msg;
```

CREATE TABLE test.kafka_msg_daily_local ON CLUSTER cluster_3s_2r ( 
  trace String, 
  app String,
  file String,
  func String,
  level String,
  msg String,
  body String,
  custom String,
  ts DateTime
) ENGINE = ReplicatedMergeTree('/clickhouse/test/tables/{layer}_{shard}/kafka_msg_daily_local', '{replica}') 
PARTITION BY toYYYYMMDD(ts) ORDER BY (trace, ts) 
SETTINGS index_granularity = 8192,storage_policy='default';



CREATE TABLE test.kafka_msg_daily_all ON CLUSTER cluster_3s_2r as test.kafka_msg_daily_local
ENGINE = Distributed('cluster_3s_2r', 'test', 'kafka_msg_daily_local', rand());


CREATE TABLE kafka_msg (
  trace String, 
  app String,
  file String,
  func String,
  level String,
  msg String,
  body String,
  custom String,
  ts DateTime
) ENGINE = Kafka SETTINGS kafka_broker_list = '10.0.2.0:9092,10.0.2.1:9092,10.0.2.2:9092',kafka_topic_list = 'app_log',kafka_group_name = 'clickhouse_group',kafka_format = 'JSONEachRow';


CREATE MATERIALIZED VIEW kafka_msg_daily_view TO kafka_msg_daily_local AS SELECT trace,app,file,func,level,msg,body,custom,ts FROM kafka_msg;