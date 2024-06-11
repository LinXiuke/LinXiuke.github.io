---

title: NameServer

date: 2024-06-06

categories:

-   RocketMQ

---

RocketMQ摒弃了业界常用的ZookKeeper作为注册中心，而是自研NameServer作为注册中心实现元数据的管理（topic路由信息等）。topic路由消息无须在集群之间保持强一致，而是追求最终一致性，并且能容忍分钟级的不一致。RocketMQ的NameServer集群之间互不通信，降低了NameServer实现的复杂度，对网络的要求也降低了，性能相比ZooKeeper有了极大提升。
<!--more-->
## 路由元信息

先看一下NameServer存储了哪些信息，具体信息见RocketMQ源码RouteInfoManager类。

```java
	// topic消息队列的路由信息，消息发送时根据路径由表进行负债均衡
    private final Map<String/* topic */, Map<String, QueueData>> topicQueueTable;

    // Broker基础信息，包括brokerName、所属集群名称、主备Broker地址
    private final Map<String/* brokerName */, BrokerData> brokerAddrTable;

    // Broker集群信息，存储集群中所有Broker的名称
    private final Map<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;

    // Broker状态信息，NameServer每次收到心跳包会替换该信息
    private final Map<BrokerAddrInfo/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;

    // Broker上的FilterServer列表，用于类模式消息过滤
    private final Map<BrokerAddrInfo/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

QueueData信息如下：

```java
    private String brokerName;
    private int readQueueNums;
    private int writeQueueNums;
    private int perm;
    private int topicSysFlag;
```

BrokerData信息如下：

```java
	
    private String cluster;
    private String brokerName;

    /**
     * The container that store the all single instances for the current broker replication cluster.
     * The key is the brokerId, and the value is the address of the single broker instance.
     */
    private HashMap<Long, String> brokerAddrs;
```

BrokerLiveInfo信息如下：

```java
    private long lastUpdateTimestamp;
    private long heartbeatTimeoutMillis;
    private DataVersion dataVersion;
    private Channel channel;
    private String haServerAddr;
```

> RocketMQ基于发布订阅模式，一个topic有多个消息队列，一个Broker默认为每个主题创建4个读队列和4个写队列。多个Broker组成一个集群，BrokerName由相同的多台Broker组成主从架构，brokerId=0代表主节点。

## 路由注册

### 概要

RockeMQ路由注册是通过Broker与NameServer的心跳功能实现的。Broker启动时向集群中所有的NameServer发送心跳语句，每个30s向集群中所有的NameServer发送心跳包，NameServer收到心跳包时会先更新brokerLiveTable缓存中的BrokerLiveInfo的lastUpdateTimestamp，然后每隔10s扫描一个brokerLiveTable，如果连续120s没有收到心跳包，NameServer将移除该Broker的路由信息，同时关闭Socket连接。



### 流程

NameServer处理心跳包流程：

1.  路由注册需要加锁，防止并发修改RouteInfoManager中的路由表。首先判断Broker所属集群是否存在，如果不存在则创建，然后将Broker名加入集群Broker集合。
2.  维护BrokerData信息，首先从BrokerAddrTable中根据broker名尝试获取Broker信息，如果不存在则新建BrokerData并放入brokerAddrTable，registerFrist设置为true；如果存在，直接替换原先的Broker信息，registerFirst设置为false，表示非第一次注册。
3.  如果Broker为主节点，并且Broker的topic配置信息发生变化或者是初次注册，则需要创建活更新topic路由元数据，并填充topicQueueTable，其实就是为默认主题自动注册路由信息。RouteInfoManager#createAndUpdateQueueData。
4.  更新BrokerLiveInfo，存储状态正常的Broker信息表。
5.  注册Broker的过滤器Servrver地址列表，一个Broker上会关联多个FilterServer消息过滤服务器。如果此Broker为从节点则需要查找该Broker的主节点信息，并更新对应的masterAddr属性。

### 设计亮点

NameServer和Broker保持长连接，Broker的状态信息存储在brokerLiveTable中，NameServer每收到一个心跳包，将更新brokerLiveTable中关于Broker的状态信息以及路由表（topicQueueTable、brokerAddrTable、brokerLiveTable、filterServerTable）。这些路由表实际用的是ConcurrentHashMap，允许多个消息发送者并发读操作，保证消息发送时的高并发。
