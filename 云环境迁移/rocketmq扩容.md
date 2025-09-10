# rocketmq扩容(4.3.0)

自建 **RocketMQ 集群扩容** 可以分两种思路：

------

## 1. 横向扩容（加 Broker / NameServer 实例）

### 添加新的 NameServer

- NameServer 无状态，直接多开几个进程/容器，客户端会自动感知。
- 推荐至少 2–3 个 NameServer 保证高可用。

### 添加新的 Broker

1. **准备机器 / 容器**，安装 jdk,RocketMQ。

2. 新建 `broker.conf`，示例：

   ```
   brokerClusterName=DefaultCluster
   brokerName=broker-b
   brokerId=0   # 0 表示 Master，大于 0 表示 Slave
   namesrvAddr=ip1:9876;ip2:9876
   listenPort=10911
   storePathRootDir=/data/rocketmq/store
   storePathCommitLog=/data/rocketmq/store/commitlog
   ```

3. 启动 broker：

   ```
   sh bin/mqbroker -c conf/broker.conf -n ip1:9876;ip2:9876 &
   ```

4. NameServer 自动感知新 Broker，客户端会在下一次路由刷新时获取新信息。

------

## 2. Topic 扩容（增加队列数）

Broker 增多以后，单个 Topic 的队列数可能不足以均匀分摊。用 `mqadmin` 调整：

```
sh bin/mqadmin updateTopic \
  -n ip1:9876;ip2:9876 \
  -t YourTopic \
  -c DefaultCluster \
  -r 8 -w 8   # 分别指定读队列和写队列数
```

队列数一旦增加，新的消息会分布到更多队列，消费者端会触发 rebalance。

------

## 3. Broker 主从模式选择

- **传统主从**：`brokerId=0` 是 Master，同一个 `brokerName` 下 `brokerId>0` 的是 Slave。
- **DLedger 模式**（推荐）：基于 Raft 协议，自动选主，支持更好的高可用和自动故障转移。文档在 RocketMQ DLedger。

------

## 4. 注意事项

- **消费者负载均衡**：RocketMQ 消费者会根据队列数和消费组大小自动 rebalance，扩容后要观察消费是否均衡。

- **消息堆积**：扩容不会自动迁移老消息，老队列里的消息还是要在原 Broker 上消费完。

- **监控**：关注 `broker.log`、磁盘 IO、PageCache 命中率和 JVM GC 情况。

- **版本**：RocketMQ 4.x 与 5.x 在 Controller / DLedger 支持上差别很大，建议新集群直接走 5.x + DLedger。

  

  

  ## 操作：

  ## 🚀 RocketMQ 自建集群扩容操作剧本

  ## 一、前置准备

  1. **确认现有集群信息**
     - NameServer 节点数（推荐 ≥ 2）
     - Broker 集群拓扑：`ClusterName`、Broker 数量、Topic 队列分布
     - 当前消息积压情况（用 `mqadmin consumerProgress` 查看）
  2. **准备新节点机器**
     - 配置：CPU ≥ 4C、内存 ≥ 8G、磁盘 SSD/高速云盘
     - 端口：10911（Broker）、10909（HA）、9876（NameServer）开放
     - 网络：和现有 NameServer、Broker 节点互通
  3. **同步部署包**
     - RocketMQ 版本保持一致
     - 配置 JDK 8/11（RocketMQ 4.x 支持 JDK8/11，5.x 更推荐 11+）

  ------

  ## 二、扩容 NameServer（可选）

  1. 启动命令：

     ```
     nohup sh bin/mqnamesrv &
     ```

  2. 在现有集群客户端配置里，追加新的 NameServer 地址：

     ```
     namesrvAddr=ns1:9876;ns2:9876;ns3:9876
     ```

  3. 验证：

     ```
     sh bin/mqadmin clusterList -n ns1:9876
     ```

  ------

  ## 三、扩容 Broker

  ### 1. 配置新 Broker

  新建 `conf/broker-b.conf`：

  ```
  brokerClusterName=DefaultCluster
  brokerName=broker-b          # 新的 Broker 名
  brokerId=0                   # 0 = Master
  deleteWhen=04
  fileReservedTime=48
  namesrvAddr=ns1:9876;ns2:9876
  listenPort=10911
  storePathRootDir=/data/rocketmq/store
  storePathCommitLog=/data/rocketmq/store/commitlog
  autoCreateTopicEnable=true
  ```

  ### 2. 启动新 Broker

  ```
  nohup sh bin/mqbroker -c conf/broker-b.conf &
  ```

  ### 3. 验证是否注册成功

  ```
  sh bin/mqadmin clusterList -n ns1:9876
  ```

  能看到新 `broker-b` 即成功。

  ------

  ## 四、调整 Topic 队列数（Topic 扩容）

  1. 修改队列数：

  ```
  sh bin/mqadmin updateTopic \
    -n ns1:9876;ns2:9876 \
    -t TestTopic \
    -c DefaultCluster \
    -r 8 -w 8
  ```

  这里 `-r` 是读队列数，`-w` 是写队列数。建议设置为 broker 数 × 每 broker 队列数。

  1. 验证：

  ```
  sh bin/mqadmin topicRoute -n ns1:9876 -t TestTopic
  ```

  可以看到新 broker 已有队列分配。

  ------

  ## 五、消费者端注意事项

  - 消费者会在 **rebalance** 时感知新的队列并开始消费。

  - 确认消费进度：

    ```
    sh bin/mqadmin consumerProgress -n ns1:9876
    ```

  - 老消息不会迁移，会在老 Broker 上消费完，新消息才会流入新队列。

  ------

  ## 六、监控与优化

  - **日志检查**：`~/logs/rocketmqlogs/` 下的 broker.log、namesrv.log
  - **性能指标**：磁盘 I/O、page cache 命中率、GC 日志
  - **告警点**：消息堆积、Consumer rebalance 卡住、Broker 注册失败

  ------

  ## 七、可选：DLedger 高可用扩容（RocketMQ 5.x 推荐）

  - 替代传统 Master-Slave，基于 Raft 协议，自动选主。
  - 新增节点时，配置 `enableDLegerCommitLog=true`，并在 `dLegerPeers` 写上所有成员地址。
  - 新 Broker 加入时会自动复制日志，数据安全性更高。

  ------

  ✅ **最终效果**：

  - NameServer 增加 → 路由更可靠
  - Broker 增加 → 吞吐量更高
  - Topic 队列增加 → 消费分摊更均衡

  平滑迁移扩容参考链接





启动console：

