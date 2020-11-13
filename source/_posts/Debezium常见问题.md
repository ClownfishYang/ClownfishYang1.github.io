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
如果使用的其他模式，那么只能手动修改记录的偏移量，虽然官网提供了[how_to_change_the_offsets_of_the_source_database](https://debezium.io/documentation/faq/#how_to_change_the_offsets_of_the_source_database)的教程，但修改起来还是很麻烦。

手动修改偏移量：
```
# 找到offset.storage的配置(kafka-connector配置文件)
offset.storage.topic=realtime-connect-offsets
offset.storage.replication.factor=3
offset.storage.partitions=2

# 找到bootstrap.servers
bootstrap.servers=localhost:9092,localhost:9093,localhost:9094

# offset.storage.topic中的消息格式
["debezium connector name",{"server":"database.server.name"}], {"ts_sec":当前时间戳,"file":"binlog fileName","pos":偏移量,"row":1,"server_id":server id,"event":2}

# 向offset.storage.topic中生产一条消息
# 因为消息格式为key:value，运行producer时必须加上parse.key和key.separator属性
kafka-console-producer.sh --broker-list localhost:9092,localhost:9093,localhost:9094 --topic topic-name --property "parse.key=true" --property "key.separator=:"
# producer启动后生产消息
[\"mysql_test_name\",{\"server\":\"mysql_test\"}]:{\"ts_sec\":1605149663000,\"file\":\"binlog.000560\",\"pos\":232604132,\"row\":1,\"server_id\":572,\"event\":2}
```
修改完成后，重新启动connector就会从修改的binlog pos开始读取，但是中间的数据依旧会丢失。

### Mysql 主库磁盘不足导致binlog不完整
出现binlog文件被删除的告警错误，发现Connector停止时间才八个小时，而binlog存储的还有24小时，错误信息大致如下，调整了部分信息。
```
[2020-11-11 15:15:20,692] INFO WorkerSourceTask{id=mysql_connector_test_1-0} Committing offsets (org.apache.kafka.connect.runtime.WorkerSourceTask:416)
[2020-11-11 15:15:20,693] INFO WorkerSourceTask{id=mysql_connector_test_1-0} flushing 0 outstanding messages for offset commit (org.apache.kafka.connect.runtime.WorkerSourceTask:433)
[2020-11-11 15:15:20,693] WARN Couldn't commit processed log positions with the source database due to a concurrent connector shutdown or restart (io.debezium.connector.common.BaseSourceTask:238)
[2020-11-11 15:15:20,709] INFO WorkerSourceTask{id=mysql_connector_test_2-0} Committing offsets (org.apache.kafka.connect.runtime.WorkerSourceTask:416)
[2020-11-11 15:15:20,709] INFO WorkerSourceTask{id=mysql_connector_test_2-0} flushing 0 outstanding messages for offset commit (org.apache.kafka.connect.runtime.WorkerSourceTask:433)
[2020-11-11 15:15:20,709] WARN Couldn't commit processed log positions with the source database due to a concurrent connector shutdown or restart (io.debezium.connector.common.BaseSourceTask:238)
[2020-11-11 15:15:21,605] INFO [Consumer clientId=mysql_connector_test_2-dbhistory, groupId=mysql_connector_test_2-dbhistory] Revoke previously assigned partitions ddlhistory.sync_36-0 (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:286)
[2020-11-11 15:15:21,605] INFO [Consumer clientId=mysql_connector_test_2-dbhistory, groupId=mysql_connector_test_2-dbhistory] Member mysql_connector_test_2-dbhistory-4acebc73-3006-48f7-99d1-0c80976ba224 sending LeaveGroup request to coordinator 172.26.77.37:9092 (id: 2147483644 rack: null) due to the consumer is being closed (org.apache.kafka.clients.consumer.internals.AbstractCoordinator:916)
[2020-11-11 15:15:21,610] INFO Finished database history recovery of 5009 change(s) in 9828 ms (io.debezium.relational.history.DatabaseHistoryMetrics:119)
[2020-11-11 15:15:21,931] INFO Step 0: Get all known binlogs from MySQL (io.debezium.connector.mysql.MySqlConnectorTask:551)
[2020-11-11 15:15:21,936] INFO MySQL has the binlog file 'binlog.000247' required by the connector (io.debezium.connector.mysql.MySqlConnectorTask:570)
[2020-11-11 15:15:21,963] INFO Requested thread factory for connector MySqlConnector, id = sync_36 named = binlog-client (io.debezium.util.Threads:270)
[2020-11-11 15:15:22,030] INFO Creating thread debezium-mysqlconnector-sync_36-binlog-client (io.debezium.util.Threads:287)
[2020-11-11 15:15:22,034] INFO Creating thread debezium-mysqlconnector-sync_36-binlog-client (io.debezium.util.Threads:287)
[2020-11-11 15:15:22,169] INFO Connected to MySQL binlog at mysql:3306, starting at binlog file 'binlog.000247', pos=218926431, skipping 3 events plus 1 rows (io.debezium.connector.mysql.BinlogReader:1111)
[2020-11-11 15:15:22,170] INFO Creating thread debezium-mysqlconnector-sync_36-binlog-client (io.debezium.util.Threads:287)
[2020-11-11 15:15:22,171] INFO Waiting for keepalive thread to start (io.debezium.connector.mysql.BinlogReader:412)
[2020-11-11 15:15:22,172] INFO Keepalive thread is running (io.debezium.connector.mysql.BinlogReader:419)
[2020-11-11 15:15:22,273] INFO WorkerSourceTask{id=mysql_connector_test_2-0} Source task finished initialization and start (org.apache.kafka.connect.runtime.WorkerSourceTask:209)
[2020-11-11 15:15:24,994] INFO [Consumer clientId=mysql_connector_test_1-dbhistory, groupId=mysql_connector_test_1-dbhistory] Revoke previously assigned partitions ddlhistory.sync_39-0 (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:286)
[2020-11-11 15:15:24,994] INFO [Consumer clientId=mysql_connector_test_1-dbhistory, groupId=mysql_connector_test_1-dbhistory] Member mysql_connector_test_1-dbhistory-3f3de8fa-2a48-473f-9484-b7d323b55972 sending LeaveGroup request to coordinator 172.26.77.110:9092 (id: 2147483645 rack: null) due to the consumer is being closed (org.apache.kafka.clients.consumer.internals.AbstractCoordinator:916)
[2020-11-11 15:15:24,998] INFO Finished database history recovery of 8384 change(s) in 13322 ms (io.debezium.relational.history.DatabaseHistoryMetrics:119)
[2020-11-11 15:15:25,102] INFO Step 0: Get all known binlogs from MySQL (io.debezium.connector.mysql.MySqlConnectorTask:551)
[2020-11-11 15:15:25,107] INFO Connector requires binlog file 'binlog.000559', but MySQL only has binlog.000384, binlog.000385, binlog.000386, binlog.000387, binlog.000388, binlog.000389, binlog.000390, binlog.000391, binlog.000392, binlog.000393, binlog.000394, binlog.000395, binlog.000396, binlog.000397, binlog.000398, binlog.000399, binlog.000400, binlog.000401, binlog.000402, binlog.000403, binlog.000404, binlog.000405, binlog.000406, binlog.000407, binlog.000408, binlog.000409, binlog.000410, binlog.000411, binlog.000412, binlog.000413, binlog.000414, binlog.000415, binlog.000416, binlog.000417, binlog.000418, binlog.000419, binlog.000420, binlog.000421, binlog.000422, binlog.000423, binlog.000424, binlog.000425, binlog.000426, binlog.000427, binlog.000428, binlog.000429, binlog.000430, binlog.000431, binlog.000432, binlog.000433, binlog.000434, binlog.000435, binlog.000436, binlog.000437, binlog.000438, binlog.000439, binlog.000440, binlog.000441, binlog.000442, binlog.000443, binlog.000444, binlog.000445, binlog.000446, binlog.000447, binlog.000448, binlog.000449, binlog.000450, binlog.000451, binlog.000452, binlog.000453, binlog.000454, binlog.000455, binlog.000456, binlog.000457, binlog.000458, binlog.000459, binlog.000460, binlog.000461, binlog.000462, binlog.000463, binlog.000464, binlog.000465, binlog.000466, binlog.000467, binlog.000468, binlog.000469, binlog.000470, binlog.000471, binlog.000472, binlog.000473, binlog.000474, binlog.000475, binlog.000476, binlog.000477, binlog.000478, binlog.000479, binlog.000480, binlog.000481, binlog.000482, binlog.000483, binlog.000484, binlog.000485, binlog.000486, binlog.000487, binlog.000488, binlog.000489, binlog.000490, binlog.000491, binlog.000492, binlog.000493, binlog.000494, binlog.000495, binlog.000496, binlog.000497, binlog.000498, binlog.000499, binlog.000500, binlog.000501, binlog.000502, binlog.000503, binlog.000504, binlog.000505, binlog.000506, binlog.000507, binlog.000508, binlog.000509, binlog.000510, binlog.000511, binlog.000512, binlog.000513, binlog.000514, binlog.000515, binlog.000516, binlog.000517, binlog.000518, binlog.000519, binlog.000520, binlog.000521, binlog.000522, binlog.000523, binlog.000524, binlog.000525, binlog.000526, binlog.000527, binlog.000528, binlog.000529, binlog.000530, binlog.000531, binlog.000532, binlog.000533, binlog.000534, binlog.000535, binlog.000536, binlog.000537, binlog.000538, binlog.000539, binlog.000540, binlog.000541, binlog.000542, binlog.000543, binlog.000544, binlog.000545, binlog.000546, binlog.000547, binlog.000548, binlog.000549, binlog.000550, binlog.000551, binlog.000552, binlog.000553, binlog.000554, binlog.000555, binlog.000556, binlog.000557, binlog.000558 (io.debezium.connector.mysql.MySqlConnectorTask:566)
[2020-11-11 15:15:25,108] INFO Stopping down connector (io.debezium.connector.common.BaseSourceTask:187)
[2020-11-11 15:15:25,108] INFO Stopping MySQL connector task (io.debezium.connector.mysql.MySqlConnectorTask:458)
[2020-11-11 15:15:25,108] INFO WorkerSourceTask{id=mysql_connector_test_1-0} Committing offsets (org.apache.kafka.connect.runtime.WorkerSourceTask:416)
[2020-11-11 15:15:25,108] INFO WorkerSourceTask{id=mysql_connector_test_1-0} flushing 0 outstanding messages for offset commit (org.apache.kafka.connect.runtime.WorkerSourceTask:433)
[2020-11-11 15:15:25,109] ERROR WorkerSourceTask{id=mysql_connector_test_1-0} Task threw an uncaught and unrecoverable exception (org.apache.kafka.connect.runtime.WorkerTask:179)
org.apache.kafka.connect.errors.ConnectException: The connector is trying to read binlog starting at binlog file 'binlog.000559', pos=271193190, skipping 3 events plus 1 rows, but this is no longer available on the server. Reconfigure the connector to use a snapshot when needed.
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
[2020-11-11 15:15:25,110] ERROR WorkerSourceTask{id=mysql_connector_test_1-0} Task is being killed and will not recover until manually restarted (org.apache.kafka.connect.runtime.WorkerTask:180)
[2020-11-11 15:15:25,110] INFO Connector has already been stopped (io.debezium.connector.common.BaseSourceTask:179)
[2020-11-11 15:15:25,110] INFO [Producer clientId=connector-producer-mysql_connector_test_1-0] Closing the Kafka producer with timeoutMillis = 30000 ms. (org.apache.kafka.clients.producer.KafkaProducer:1183)[2020-11-11 15:15:30,697] INFO WorkerSourceTask{id=mysql_connector_test_1-0} Committing offsets (org.apache.kafka.connect.runtime.WorkerSourceTask:416)
```
有个关键的信息是，Debezium保存的是binlog.000559，而主库却只有385-558的binlog，猜测可能跟binlog清理设置有关系。  
将快照模式依次调整为`schema_only_recovery`和`when_needed`都报出了相同的错误：
```
2020-11-12 01:51:06,372] ERROR WorkerSourceTask{id=mysql_connector_test_1-0} Task threw an uncaught and unrecoverable exception (org.apache.kafka.connect.runtime.WorkerTask:179)
org.apache.kafka.connect.errors.ConnectException: binlog truncated in the middle of event; consider out of disk space on master; the first event 'binlog.000559' at 271193190, the last event read from '/data/dbdata/mysqllog/binlog/binlog.000559' at 271193190, the last
 byte read from '/data/dbdata/mysqllog/binlog/binlog.000559' at 271193209. Error code: 1236; SQLSTATE: HY000.
        at io.debezium.connector.mysql.AbstractReader.wrap(AbstractReader.java:230)
        at io.debezium.connector.mysql.AbstractReader.failed(AbstractReader.java:196)
        at io.debezium.connector.mysql.BinlogReader$ReaderThreadLifecycleListener.onCommunicationFailure(BinlogReader.java:1125)
        at com.github.shyiko.mysql.binlog.BinaryLogClient.listenForEventPackets(BinaryLogClient.java:985)
        at com.github.shyiko.mysql.binlog.BinaryLogClient.connect(BinaryLogClient.java:581)
        at com.github.shyiko.mysql.binlog.BinaryLogClient$7.run(BinaryLogClient.java:860)
        at java.lang.Thread.run(Thread.java:745)
Caused by: com.github.shyiko.mysql.binlog.network.ServerException: binlog truncated in the middle of event; consider out of disk space on master; the first event 'binlog.000559' at 271193190, the last event read from '/data/dbdata/mysqllog/binlog/binlog.000559' at 27
1193190, the last byte read from '/data/dbdata/mysqllog/binlog/binlog.000559' at 271193209.
        at com.github.shyiko.mysql.binlog.BinaryLogClient.listenForEventPackets(BinaryLogClient.java:949)
        ... 3 more
[2020-11-12 01:51:06,372] ERROR WorkerSourceTask{id=mysql_connector_test_1-0} Task is being killed and will not recover until manually restarted (org.apache.kafka.connect.runtime.WorkerTask:180)
[2020-11-12 01:51:06,372] INFO Stopping down connector (io.debezium.connector.common.BaseSourceTask:187)
[2020-11-12 01:51:06,372] INFO Stopping MySQL connector task (io.debezium.connector.mysql.MySqlConnectorTask:458)
```
主库是存在binlog.000559这个文件的，但是可能由于主库磁盘存储不足导致日志记录不完整，被截断了。
通过[MySQL Replication: ‘Got fatal error 1236’ causes and cures](https://www.percona.com/blog/2014/10/08/mysql-replication-got-fatal-error-1236-causes-and-cures/)
和[Got fatal error 1236原因](http://blog.itpub.net/22664653/viewspace-1714269/)了解到，只能将binlog指向下一个可用的binlog file，所以只能通过`手动修改偏移量`的方式解决。

## 参考文章
[<i class="fas fa-paperclip"></i> debezium documentation](https://debezium.io/documentation/reference/1.3/connectors/mysql.html)
[<i class="fas fa-paperclip"></i> debezium unable to start if the binlog is purged](https://groups.google.com/g/debezium/c/di3jWxMzq9c)
[<i class="fas fa-paperclip"></i> Resume Debezium MySQL connector](https://gitter.im/debezium/user?at=5d1dde6626206b667c89e0c4)
[<i class="fas fa-paperclip"></i> how_to_change_the_offsets_of_the_source_database](https://debezium.io/documentation/faq/#how_to_change_the_offsets_of_the_source_database)