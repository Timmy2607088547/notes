# Mq迁移

### pre环境限制网段访问，要么去指定服务器访问mq控制台，或者本地配置apisix的负载均衡访问虚拟ip



## 1.mqcloud介绍

  MQCloud是[RocketMQ](https://github.com/apache/rocketmq)的管理平台，它具备以下特性：

- 一，跨集群：同时管理多个集群，对使用者透明。
- 二，预警功能：针对消费堆积，失败，异常等情况预警，处理。
- 三，简单明了：使用者视图:对拓扑，流量，消费状况等指标进行直接展示；管理员视图:集群维护监控，流程审批等。
- 四，安全：用户隔离，操作审批，数据安全。

## 2.使用规范

### 一、命名规范

- topic命名规范：业务名-{环境}-topic 。
- producer命名规范：{topic名}-producer 。 不同业务线往同一topic发消息，加上业务前缀区分，mall-search-{topic名}-producer
- consumer命名规范：{topic名}-consumer 。不同业务线消费同一topic发消息，加上业务前缀区分，mall-search-{topic名}-consumer

命名尽量做到**见名知意**，同时，全局唯一，topic ，producer， consumer，名称均不重复为宜。

> ```
> 要求：部署多套测试环境时，{环境}依次命名为stage1、stage2、stage3
> ```

### 二、使用规范

1. topic申请：

 topic的数量对MQ集群几乎没有任何影响，但是项目中topic的数据增多，会增加维护成本，代码冗余度高。尽量减少无用的topic申请。

> 要求: 同一业务线，相似业务应该共用一个topic，消息体中加入type字段区分消息类型。
>
> 反例：ERP审核、设备挂应收、确认收货等业务，分别申请了不同topic，不符合精简原则

 

   \2. 消息内容：

严禁使用自定义对象当做消息体，非JDK 提供的对象严禁使用，推荐使用 java.util.Map 。

原因：MQCloud提供的客户端，增加了自己封装的 序列化与反序列化操作。

   \3. 日志规范

​    生产者或消费者 逻辑处理前需要记录消息的 msgId ，以便回查问题。视业务情况打印消息内容。

### 三、部署

使用rocketmq4.3.0版本整体采用2台机器

部署模式采用 master->slave结构，对2台机器进行交叉部署，具体如下

2台机器同时为nameserver



| namserver  | IP:Port   |
| :--------- | :-------- |
| namersrv-1 | xxxx:9898 |
| namesrv-2  | xxxx:9898 |

Broker采用master和slave交叉部署：

| Broker-name | ip     | 角色   |
| :---------- | :----- | :----- |
| broker-a    | :10915 | master |
| broker-b    | :10920 | master |
| broker-a-s  | :10915 | slave  |
| broker-b-s  | :10920 | slave  |

### broker配置

> ```yaml
> #集群名字
> brokerClusterName=mq-cluster
> #broker名字
> brokerName=broker-a
> 角色ID0位master，大于0位slave
> brokerId=``0
> 内网IP，如果服务器有多个IP，这个是必须配置的，否则会报错
> brokerIP1=xxxx
> 每天删除日志时间
> deleteWhen=04
> 文件保存时间
> fileReservedTime=168
> 角色
> brokerRole=ASYNC_MASTER
> 磁盘写入方式
> flushDiskType=ASYNC_FLUSH
> 监听端口
> listenPort=10915
> 是否可以自动创建topic
> autoCreateTopicEnable=true
> clusterTopicEnable=true
> autoCreateSubscriptionGroup=true
> storePathRootDir=/data/soft/rocketmq-4.3.0/broker-a/data
> storePathCommitLog=``//data/soft/rocketmq-4.3.0/broker-a/data/commitlog
> storePathIndex=/data/soft/rocketmq-4.3.0/broker-a/data/index
> storeCheckpoint=/data/soft/rocketmq-4.3.0/broker-a/data/checkpoint
> abortFile=/data/soft/rocketmq-4.3.0/broker-a/data/abort
> namesrv集群列表
> namesrvAddr=xxx
> ```