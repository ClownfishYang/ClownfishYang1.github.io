---
title: Debezium常见问题
date: 2020-11-05 11:17:25
tags:
- Debezium
- Mysql
- binlog
- 实时同步
categories:
- 技术
---
记录Debezium使用过程中遇到的一些常见问题，后续也会持续更新。
<!--more-->

## 常见问题

### Mysql binlog 文件清理
Debezium 同步Mysql依赖于binlog文件，`Debezium Mysql`会将binlog事件转换为Debezium更改事件并将这些事件记录到Kafka中，这就是`Debezium Mysql`的快照(snapshots)。
当connector 异常重启后，会执行[快照恢复](https://debezium.io/documentation/reference/1.3/connectors/mysql.html#how-the-mysql-connector-performs-database-snapshots_debezium)：
1. 以可重复的读取语义启动事务，确保针对一致性快照中完成事务的所有后续读取操作；
2. 读取当前serverName记录binlog的偏移量；
3. 从偏移量位置开始读取允许的数据库和表的schema语句；
4. 将DDL修改写入到schema修改的topic；
5. 扫描数据表，并为每行数据创建相关联的topic；
6. 提交事务，释放锁；
7. 记录事务完成的偏移量到快照；

当`Debezium Mysql Connector`停止时间过长导致binlog文件过期被清理或者磁盘紧张被清理时，Connector 记录的最后偏移量可能会丢失。
主要异常信息如下：
```
[2020-11-04 10:29:21,539] INFO Kafka version: 2.4.1 (org.apache.kafka.common.utils.AppInfoParser:117)
[2020-11-04 10:29:21,539] INFO Kafka commitId: c57222ae8cd7866b (org.apache.kafka.common.utils.AppInfoParser:118)
[2020-11-04 10:29:21,539] INFO Kafka startTimeMs: 1604485761539 (org.apache.kafka.common.utils.AppInfoParser:119)
[2020-11-04 10:29:21,540] INFO [Consumer clientId=dbhistory, groupId=dbhistory] Subscribed to topic(s): dbhistory_topic (org.apache.kafka.clients.consumer.KafkaConsumer:969)
[2020-11-04 10:29:21,546] INFO [Consumer clientId=dbhistory, groupId=dbhistory] Cluster ID: erxHIDoJRXGAjZ0AujKkKA (org.apache.kafka.clients.Metadata:259)
[2020-11-04 10:29:21,547] INFO Database history topic 'dbhistory_topic' has correct settings (io.debezium.relational.history.KafkaDatabaseHistory:440)
[2020-11-04 10:29:21,548] INFO [Consumer clientId=dbhistory, groupId=dbhistory] Discovered group coordinator 172.26.77.29:9092 (id: 2147483646 rack: null) (org.apache.kafka.clients.consumer.internals.AbstractCoordinator:756)
[2020-11-04 10:29:21,549] INFO [Consumer clientId=dbhistory, groupId=dbhistory] (Re-)joining group (org.apache.kafka.clients.consumer.internals.AbstractCoordinator:533)
[2020-11-04 10:29:21,552] INFO [Consumer clientId=dbhistory, groupId=dbhistory] (Re-)joining group (org.apache.kafka.clients.consumer.internals.AbstractCoordinator:533)
[2020-11-04 10:29:21,555] INFO [Consumer clientId=dbhistory, groupId=dbhistory] Finished assignment for group at generation 1: {dbhistory-b9169794-6a9c-4d16-abb2-70f291ffe28a=Assignment(partitions=[dbhistory_topic-0])} (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:585)
[2020-11-04 10:29:21,556] INFO [Consumer clientId=dbhistory, groupId=dbhistory] Successfully joined group with generation 1 (org.apache.kafka.clients.consumer.internals.AbstractCoordinator:484)
[2020-11-04 10:29:21,557] INFO [Consumer clientId=dbhistory, groupId=dbhistory] Adding newly assigned partitions: dbhistory_topic-0 (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:267)
[2020-11-04 10:29:21,558] INFO [Consumer clientId=dbhistory, groupId=dbhistory] Found no committed offset for partition dbhistory_topic-0 (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:1241)
[2020-11-04 10:29:21,559] INFO [Consumer clientId=dbhistory, groupId=dbhistory] Resetting offset for partition dbhistory_topic-0 to offset 0. (org.apache.kafka.clients.consumer.internals.SubscriptionState:381)
[2020-11-04 10:29:22,105] INFO [Worker clientId=connect-1, groupId=realtime-cluster] Tasks [49_gb_soa_prd_mysql_gb_s01.aws-virginia-1.hq_3306_obs_gb-0] configs updated (org.apache.kafka.connect.runtime.distributed.DistributedHerder:1411)
[2020-11-04 10:29:22,105] INFO [Worker clientId=connect-1, groupId=realtime-cluster] Handling task config update by restarting tasks [49_gb_soa_prd_mysql_gb_s01.aws-virginia-1.hq_3306_obs_gb-0] (org.apache.kafka.connect.runtime.distributed.DistributedHerder:574)
[2020-11-04 10:29:22,105] INFO Stopping task 49_gb_soa_prd_mysql_gb_s01.aws-virginia-1.hq_3306_obs_gb-0 (org.apache.kafka.connect.runtime.Worker:704)
[2020-11-04 10:29:24,094] INFO [Consumer clientId=dbhistory, groupId=dbhistory] Revoke previously assigned partitions dbhistory_topic-0 (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:286)
[2020-11-04 10:29:24,094] INFO [Consumer clientId=dbhistory, groupId=dbhistory] Member dbhistory-b9169794-6a9c-4d16-abb2-70f291ffe28a sending LeaveGroup request to coordinator 172.26.77.29:9092 (id: 2147483646 rack: null) due to the consumer is being closed (org.apache.kafka.clients.consumer.internals.AbstractCoordinator:916)
[2020-11-04 10:29:24,096] INFO Finished database history recovery of 3548 change(s) in 2559 ms (io.debezium.relational.history.DatabaseHistoryMetrics:119)
[2020-11-04 10:29:24,099] INFO Step 0: Get all known binlogs from MySQL (io.debezium.connector.mysql.MySqlConnectorTask:551)
[2020-11-04 10:29:24,100] INFO Connector requires binlog file 'binlog.000602', but MySQL only has binlog.000998, binlog.000999, binlog.001000, binlog.001001, binlog.001002, binlog.001003, binlog.001004, binlog.001005, binlog.001006, binlog.001007, binlog.001008, binlog.001009, binlog.001010, binlog.001011, binlog.001012, binlog.001013, binlog.001014, binlog.001015, binlog.001016 (io.debezium.connector.mysql.MySqlConnectorTask:566)
[2020-11-04 10:29:24,101] INFO Stopping down connector (io.debezium.connector.common.BaseSourceTask:187)
[2020-11-04 10:29:24,101] INFO Stopping MySQL connector task (io.debezium.connector.mysql.MySqlConnectorTask:458)
[2020-11-04 10:29:24,101] INFO WorkerSourceTask{id=49_gb_soa_prd_mysql_gb_s01.aws-virginia-1.hq_3306_obs_gb-0} Committing offsets (org.apache.kafka.connect.runtime.WorkerSourceTask:416)
[2020-11-04 10:29:24,101] INFO WorkerSourceTask{id=49_gb_soa_prd_mysql_gb_s01.aws-virginia-1.hq_3306_obs_gb-0} flushing 0 outstanding messages for offset commit (org.apache.kafka.connect.runtime.WorkerSourceTask:433)
[2020-11-04 10:29:24,101] ERROR WorkerSourceTask{id=49_gb_soa_prd_mysql_gb_s01.aws-virginia-1.hq_3306_obs_gb-0} Task threw an uncaught and unrecoverable exception (org.apache.kafka.connect.runtime.WorkerTask:179)
org.apache.kafka.connect.errors.ConnectException: The connector is trying to read binlog starting at binlog file 'binlog.000602', pos=457725550, skipping 0 events plus 0 rows, but this is no longer available on the server. Reconfigure the connector to use a snapshot when needed.
	at io.debezium.connector.mysql.MySqlConnectorTask.start(MySqlConnectorTask.java:133)
	at io.debezium.connector.common.BaseSourceTask.start(BaseSourceTask.java:101)
	at org.apache.kafka.connect.runtime.WorkerSourceTask.execute(WorkerSourceTask.java:208)
	at org.apache.kafka.connect.runtime.WorkerTask.doRun(WorkerTask.java:177)
	at org.apache.kafka.connect.runtime.WorkerTask.run(WorkerTask.java:227)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
[2020-11-04 10:29:24,102] ERROR WorkerSourceTask{id=49_gb_soa_prd_mysql_gb_s01.aws-virginia-1.hq_3306_obs_gb-0} Task is being killed and will not recover until manually restarted (org.apache.kafka.connect.runtime.WorkerTask:180)
```
从异常信息来看，确实是因为binlog文件被清理导致快照记录的偏移量是丢失了，debezium有关于该问题谈论：
1. [debezium unable to start if the binlog is purged](https://groups.google.com/g/debezium/c/di3jWxMzq9c)
2. [Resume Debezium MySQL connector](https://gitter.im/debezium/user?at=5d1dde6626206b667c89e0c4)

这问题确实比较麻烦，最简单方式是使用一个新的serverName重新建立一个新的connector，但是问题就是topicName是跟serverName生成的，那说明后续处理topic的程序就需要改动。

`Debezium Mysql`定义了几种快照模式：

| 快照模式 | 说明 |
| :---- | :---- |
| initial(默认) | 仅在没有为serverName记录偏移量时才会生产快照 |
| when_needed | 在必要时运行快照，没有偏移量或者偏移量不可用时 |
| never | 不使用快照，并且首次启动时，从binlog的起始偏移量开始读取 |
| schema_only | 不需要保留数据的一致性快照，只需要保留结构的快照 |
| schema_only_recovery | 当记录现有偏移量的topic不存在时，从当前binlog的起始偏移量开始读取恢复 |

如果使用的是`schema_only`或`schema_only_recovery`模式，能够使用`when_needed`进行恢复，但是中间丢失的binlog数据也会丢失，最好是在使用`when_needed`前先进行全量恢复使其保持数据一致性。  
如果使用的其他模式，那么只能手动修改记录的偏移量，但是修改起来很麻烦，因为没有暴露这样的接口，虽然官网提供了[how_to_change_the_offsets_of_the_source_database](https://debezium.io/documentation/faq/#how_to_change_the_offsets_of_the_source_database)的教程，但还是很麻烦。



## 参考文章
[<i class="fas fa-paperclip"></i> debezium documentation](https://debezium.io/documentation/reference/1.3/connectors/mysql.html)
[<i class="fas fa-paperclip"></i> debezium unable to start if the binlog is purged](https://groups.google.com/g/debezium/c/di3jWxMzq9c)
[<i class="fas fa-paperclip"></i> Resume Debezium MySQL connector](https://gitter.im/debezium/user?at=5d1dde6626206b667c89e0c4)
[<i class="fas fa-paperclip"></i> how_to_change_the_offsets_of_the_source_database](https://debezium.io/documentation/faq/#how_to_change_the_offsets_of_the_source_database)