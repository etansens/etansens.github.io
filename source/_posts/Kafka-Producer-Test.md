---
title: Kafka Producer Test
date: 2019-11-20 15:38:41
categories:
- 大数据
tags:
- java
- kafka

---
## 环境配置
5*(24 core, 64G men, 1T*12 disk)
CentOS6.5
JDK1.8.0_91
kafka_2.11-1.0.0
brokers:
export KAFKA_HEAP_OPTS="-Xmx8G -Xms8G"
default.replication.factor = 2
num.io.threads = 16
num.network.threads = 8
num.replica.fetchers = 2
mun.partitions = 30

## 测试用例
测试服务端变动情况下生产者的运行情况及数据表现
考察重试机制，批量模式
case 0: Preferred Replica Election
case 1: Reassign Partitions
case 2: Add Partitions
case 3: Broker Down
case 4: Add Broker

## Producer配置
### producer0:默认配置
bin/kafka-producer-perf-test.sh \
--topic test \
--num-records 6000 --record-size 100 --throughput 100 \
--print-metrics \
--producer-props bootstrap.servers=10.31.33.21:6667,10.31.33.23:6667,10.31.33.25:6667

### producer1: 增加重试
bin/kafka-producer-perf-test.sh \
--topic test \
--num-records 6000 --record-size 100 --throughput 100 \
--print-metrics \
--producer-props bootstrap.servers=10.31.33.21:6667,10.31.33.23:6667,10.31.33.25:6667 \
retries=3 \
retry.backoff.ms=1000

## 测试过程
### case0-producer0:
现象：日志报错
org.apache.kafka.common.errors.NotLeaderForPartitionException: This server is not the leader for that topic-partition.
结论：数据丢失

### case0-producer1:
现象：
[2019-11-20 13:56:51,637] WARN [Producer clientId=producer-1] Got error produce response with correlation id 560 on topic-partition test-9, retrying (2 attempts left). Error: NOT_LEADER_FOR_PARTITION (org.apache.kafka.clients.producer.internals.Sender)
结论：正常

### case1-producer0:
现象：日志报错
[2019-11-19 15:49:00,959] WARN [Producer clientId=producer-1] Received unknown topic or partition error in produce request on partition test-11. The topic/partition may not exist or the user may not have Describe access to it (org.apache.kafka.clients.producer.internals.Sender)
org.apache.kafka.common.errors.UnknownTopicOrPartitionException: This server does not host this topic-partition.
[2019-11-19 15:49:01,030] WARN [Producer clientId=producer-1] Received unknown topic or partition error in produce request on partition test-16. The topic/partition may not exist or the user may not have Describe access to it (org.apache.kafka.clients.producer.internals.Sender)
org.apache.kafka.common.errors.NotLeaderForPartitionException: This server is not the leader for that topic-partition.
result：数据丢失

### case1-producer1:
数据少量丢失

### case2-producer0，case2-producer1:
需要重启producer,才能往新partition写数据


### case3-producer0:
[2019-11-20 15:16:11,922] WARN [Producer clientId=producer-1] Got error produce response with correlation id 11898 on topic-partition test-12, retrying (4 attempts left). Error: NOT_LEADER_FOR_PARTITION (org.apache.kafka.clients.producer.internals.Sender)
[2019-11-20 15:16:12,234] WARN [Producer clientId=producer-1] Connection to node 1001 could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
丢失数据
### case3-producer1:
丢失少量数据

## 总结
尽力避免导致分区重分配的操作，暂停producer进行操作

## kafka 工具应用
### 分区重新分布，可用于增加、减少副本数，重新均衡分区分布；慎用，会导致producer丢失数据
bin/kafka-reassign-partitions.sh --zookeeper 10.31.33.29:2181 --reassignment-json-file rap.json --execute
{"version":1,
"partitions":[{"topic":"test","partition":0,"replicas":[0,1,2]}]}


### 触发leader选举
bin/kafka-preferred-replica-election.sh --zookeeper 10.31.33.29:2181 --path-to-json-file ./pre.json
{
 "partitions":
  [
    {"topic": "test", "partition": 0}
  ]
}