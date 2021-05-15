#### 日志采集--Filebeat

- 软件版本：7.6.2

  

#### 1、Filebeat的核心组件

- Input组件：负责监控日志目录的变化情况，如果发现某文件被移动、删除或有新文件被创建，会将这些信息提供给`Harvester`使用；该组件会以一定间隔扫描日志文件的变化情况，如果在间隔之间删除、创建了文件，那就不会被探测到。

- Harvester组件：负责监控日志文件内容的变化情况，逐行读取每个文件，并将新内容发送到output中。

- Prospector组件：负责管理Harvsters，并且找到所有需要进行读取的数据源。

- Spooler组件：用于将`Harvester`"收割"到的日志行发送到外部系统，如ES, Kafka, Logstash等。



#### 2、核心参数

##### 2.1 全局参数

- max_procs：设置可以同时执行的最大CPU数，默认为系统中可用逻辑CPU的数量；
- queue.mem.events：存储于内存队列的事件数，排队发送，默认4096；
- queue.mem.flush.min_events：需要小于 queue.mem.events ,增加此值可提高吞吐量 ；
- queue.mem.flush.timeout：发送事件的最长时间，如果队列中存储的事件数小于min_events，达到该值就会发送；

##### 2.2 局部参数

- scan_frequency：多长时间检测一次目标路径是否有新增文件出现，默认10s；
- max_bytes：单条日志的大小限制，默认10M；
- ignore_older：指定Filebeat忽略指定时间段以外修改的日志内容；
- close_inactive：用于控制自日志文件内容没有发生变更开始等待多久就将文件关闭，同时退出对应的Harvester，默认为5m；
- close_older：如果一个文件在某个时间段内没有发生过更新，则关闭监控的文件handle,默认1h
- clean_inactive：多久清理一次注册信息，默认关闭，时间久了，注册信息的文件会越来越大，重启filebeat会变的非常慢；
- clean_removed：是否删除无法在磁盘上找到的文件的状态；
- tail_files：这个参数用来控制当开始监控一个新文件时是否从文件末尾开始读取，默认为false；



#### 3、优化实践

##### 3.1 背景

公司有一个日志收集系统，负责全公司web端项目的行为日志收集，日均PV 20亿+，单台接收机器高峰时间QPS 2.5w+（接收机器配置：24C，32G，1.5T）；

整个数据链路为接收程序将（golang实现的）日志落，由Filebeat采集日志发送到Kafka；后面就是大数据处理了，使用的是标准的Lambda 架构，一个Flink程序消费Kafka进行实时分析，另外一个Flink将日志每隔5分钟写入Hive，用于数仓离线任务分析；

之前采集日志使用过Flume，但是线上Flume占用内存过高，故改用轻量级的Filebeat。

##### 3.2 问题

每天高峰期间，离线任务在进行日志校验的时候，发现hive里面日志总量比接收机器上面少很多，延迟高达几十分钟。通过排查（具体排查过程不细说），发现是Filebeat采集的时候数据有延迟。

由于Filebeat第一版本的配置，基本都是默认配置，许多配置在当前业务场景下存在不合理的地方，故需要对配置进行优化。

##### 3.3 优化

优化之前的配置

```yaml
max_procs: 4
queue.mem.events: 4096
queue.mem.flush.min_events: 2048
queue.mem.flush.timeout: 5s

################### logs ###################
filebeat.inputs:
- type: log
  enabled: true
  paths:
  - /opt/data/test/*.log
  fields:
    kafkatopic: bigdata_test_logs
  tail_files: true


################################### kafka ########################################
output.kafka:
  enabled: true
  hosts: ["test-kafka001:6667","test-kafka002:6667","test-kafka003:6667","test-kafka004:6667","test-kafka005:6667","test-kafka006:6667","test-kafka007:6667"]
  topic: '%{[fields][kafkatopic]}'
  partition.hash:
    reachable_only: true
  compression: gzip
  worker: 4
  max_message_bytes: 1000000
  required_acks: 1

processors:
- drop_fields:
    fields: ["prospector","input","host","agent","ecs"]
```



优化之后的配置

```yaml
#################全局配置#########################
 
#设置可以同时执行的最大CPU数，默认为系统中可用逻辑CPU的数量，目前接收机器负载不高，设置为总数量的一半
max_procs: 12
#存储于内存队列的事件数，排队发送 (默认4096)
queue.mem.events: 8192
#小于 queue.mem.events ,增加此值可提高吞吐量
queue.mem.flush.min_events: 4096
#发送事件的最长时间， 如果队列中存储的事件数小于 min_events
queue.mem.flush.timeout: 5s
 
 
##############监控配置#################
monitoring:
  enabled: true
  cluster_uuid: ************
  elasticsearch:
    hosts: ["http://pub-es-c1-node001.test.com:9200", "http://pub-es-c1-node002.test.com:9200", "http://pub-es-c1-node003.test.com:9200"]
    username: *******
    password: *******
 
 
################### 局部配置 ###################
filebeat.inputs:
- type: log
  enabled: true
  #多长时间检测一次目标路径是否有新增文件出现
  scan_frequency: 20s
  #单条日志的大小限制
  max_bytes: 20480
  #可以指定Filebeat忽略指定时间段以外修改的日志内容
  ignore_older: 1h
  #这个参数用于控制自日志文件内容没有发生变更开始等待多久就将文件关闭，同时退出对应的harvester，默认为5m
  close_inactive: 10m
  #如果一个文件在某个时间段内没有发生过更新，则关闭监控的文件handle，默认1h
  close_older: 30m
  #多久清理一次注册信息(默认关闭)
  clean_inactive: 120m
  # 立即删除无法在磁盘上找到的文件的状态
  clean_removed: true
  paths:
    - /opt/data/test/*.log
  fields:
    kafkatopic: bigdata_test_logs
    tail_files: true
 
 
################################### kafka ########################################
output.kafka:
  enabled: true
  hosts: ["test-kafka001:6667","test-kafka002:6667","test-kafka003:6667","test-kafka004:6667","test-kafka005:6667","test-kafka006:6667","test-kafka007:6667"]
  topic: '%{[fields][kafkatopic]}'
  partition.hash:
    reachable_only: true
  compression: gzip
  worker: 4
  max_message_bytes: 1000000
  required_acks: 1
 
processors:
  - drop_fields:
      fields: ["prospector","input","host","agent","ecs"]
```



优化之后的配置提高了吞吐量，同时将Filebeat的Metrics信息发送到ES，通过Kbana可以直观的看到Filebeat各项指标信息。

后记：目前线上运行稳定，暂无延迟；后面打算通过ES中的Metrics信息，做Filebeat的相关监控。



参考：https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-reference-yml.html

