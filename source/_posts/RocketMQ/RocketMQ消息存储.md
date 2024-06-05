---
title: RocketMQ消息存储

date: 2024-05-31

categories:
- RocketMQ
---

RocketMQ存储的文件主要包括CommitLog文件、ConsumeQueue文件、Index文件。RocketMQ将所有主题的消息存储在同一个文件中，确保消息发送时按顺序写文件，尽最大能力确保消息发送的高性能与高吞吐量。
<!--more-->

1.  CommitLog：消息存储，所有消息主题的消息都存储在CommitLog文件中。
2.  ConsumeQueue：消息消费队列，消息到达CommitLog文件后，将异步转发到ConsumeQueue文件中，供消息消费者消费。
3.  Index：消息索引，主要存储消息key与offset的对应关系。

## CommitLog文件

RocketMQ的所有主题的消息全部写入CommitLog文件中，所有消息按照到达顺序依次追加到CommitLog文件中，消息一旦写入，不支持修改。

RocketMQ会为每条消息引入一个身份标志：消息物理偏移量，即消息存储在文件的起始位置。也是有了物理偏移量的概念，CommitLog文件也是采用偏移量，例如第一个CommitLog文件为00000000000000000000，第二个文件为00000000001073741824。文件名长度为20位，左边补零，单个文件大小默认1GB。

## ConsumeQueue文件

消息消费模型是基于主题订阅机制的，即一个消费组是消费特定主题的消息，根据主题从CommitLog文件中检索消息，效率会很低，为了解决基于topic的消息检索问题，RocketMQ引入了ConsumeQueue文件。

ConsumeQueue文件是消息消费队列文件，是CommitLog文件基于topic的索引文件，主要用于消费者根据topic消费消息，其组织文件方式为/topic/queue/file三层组织结构，具体存储路径为\$HOME/store/consumequeue/{topic}/{queueId}/{fileName}，同一个队列中存在多个消息文件。ConsumeQueue每个条目长度固定（8字节CommitLog物理偏移量，4字节消息长度，8字节tag哈希码）

消费者根据topic、消息消费进度（ConsumeQueue逻辑偏移量），即第几个ConsumeQueue条目，这样的消费进度去访问消息，通过逻辑偏移量logicOffset\*20，即可找知道该条目的起始偏移量（ConsumeQueue文件中的偏移量），然后读取该偏移量后20字节即可得到一个条目，无须遍历ConsumeQueue文件。

## Index文件

RocketMQ支持按消息熟悉检索消息，引入了ConsumeQueue文件解决了基于topic查找消息的问题，但如果想基于消息的某一个熟悉进行查找，ConsumeQueue就做不到了。所以RocketMQ有引入了Index索引文件，实现基于文件的哈希索引。

Index文件基于物理磁盘文件实现哈希索引。Index文件由40字节的文件头，500万个哈希槽、2000万个Index条目组成，每个哈希槽4字节、每个Index条目含有20个字节，分别为4字节索引key的哈希码、8字节消息物理偏移量、4字节时间戳、4字节的前一个Index条目（哈希冲突的链表结构）。



## 内存映射

RocketMQ为了提升消息写入效率，引入了内存映射，将磁盘文件映射到内存中，一操作内存的方式操作磁盘。

在Java中可以通过FileChannel的map方法创建内存映射文件。在Linux服务器由该方法创建的文件使用的就是操作系统的页缓存（pagecache）。如果RocketMQ Broker进程异常退出，存储在页缓存的数据不会丢失，操作系统会定时将页缓存的数据持久化到磁盘，不过如果是机器断电等异常情况数据还是可能丢失。



## 刷盘策略

引入了内存映射和页缓存机制后，消息会先写入页缓存，此时还没有真正持久化。那么Broker收到消息后，是存储到页缓存就直接返回成功还是要持久化到磁盘裁返回成功。为此，RocketMQ提供了两种策略：同步刷盘、异步刷盘。

*   同步刷盘：保证消息不丢失，但是写入性能会较低。
*   异步刷盘：效率更高，消息丢失的可能性也小。Broker将消息存储到pagecache后就立即返回成功，然后开启一个异步线程定时执行FileChannel的force方法，将内存的数据定时写入磁盘，默认间隔时间500ms。



## transientStorePoolEnable机制

transientStorePoolEnable机制，即内存级别的读写分离机制。RocketMQ把消息写入pagecache，消息消费时从pagecache中读取，在高并发时pagecache压力会比较大，容易出现broker busy异常。RocketMQ通过这种机制，先把消息写入堆外内存并立即返回，然后异步将堆外内存中的数据提交到pagecache，再异步刷盘到磁盘中。因为堆外内存数据并未提交，所以消息消费时不会从这里读取，而是从pagecache读取，这样就实现了内存级别的读写分离。

这种机制的有点事实现了批量化的消息写入，缺点是如果broker进程异常退出，堆外内存数据会丢失。
