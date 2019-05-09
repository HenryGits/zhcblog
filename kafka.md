1、什么是kafka

    Kafka是分布式发布-订阅消息系统，它最初是由LinkedIn公司开发的，之后成为Apache项目的一部分。
    Kafka是一个分布式，可划分的，冗余备份的持久性的日志服务，它主要用于处理流式数据。

2、为什么要使用 kafka，为什么要使用消息队列

    缓冲和削峰：上游数据时有突发流量，下游可能扛不住，或者下游没有足够多的机器来保证冗余，kafka在中间可以起到一个缓冲的作用，把消息暂存在kafka中，下游服务就可以按照自己的节奏进行慢慢处理。
    解耦和扩展性：项目开始的时候，并不能确定具体需求。消息队列可以作为一个接口层，解耦重要的业务流程。只需要遵守约定，针对数据编程即可获取扩展能力。
    冗余：可以采用一对多的方式，一个生产者发布消息，可以被多个订阅topic的服务消费到，供多个毫无关联的业务使用。
    健壮性：消息队列可以堆积请求，所以消费端业务即使短时间死掉，也不会影响主要业务的正常进行。
    异步通信：很多时候，用户不想也不需要立即处理消息。消息队列提供了异步处理机制，允许用户把一个消息放入队列，但并不立即处理它。想向队列中放入多少消息就放多少，然后在需要的时候再去处理它们。

3、kafka中的broker 是干什么的

    broker 是消息的代理，Producers往Brokers里面的指定Topic中写消息，Consumers从Brokers里面拉取指定Topic的消息，然后进行业务处理，broker在中间起到一个代理保存消息的中转站。

4、kafka中的 zookeeper 起到什么作用，可以不用zookeeper么

    zookeeper 是一个分布式的协调组件，早期版本的kafka用zk做meta信息存储，consumer的消费状态，group的管理以及 offset的值。考虑到zk本身的一些因素以及整个架构较大概率存在单点问题，新版本中逐渐弱化了zookeeper的作用。新的consumer使用了kafka内部的group coordination协议，也减少了对zookeeper的依赖，

    但是broker依然依赖于ZK，zookeeper 在kafka中还用来选举controller 和 检测broker是否存活等等。


5、kafka follower如何与leader同步数据

    Kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制。完全同步复制要求All Alive Follower都复制完，这条消息才会被认为commit，这种复制方式极大的影响了吞吐率。而异步复制方式下，Follower异步的从Leader复制数据，数据只要被Leader写入log就被认为已经commit，这种情况下，如果leader挂掉，会丢失数据，kafka使用ISR的方式很好的均衡了确保数据不丢失以及吞吐率。
    Follower可以批量的从Leader复制数据，而且Leader充分利用磁盘顺序读以及send file(zero copy)机制，这样极大的提高复制性能，内部批量写磁盘，大幅减少了Follower与Leader的消息量差。

6、什么情况下一个 broker 会从 isr中踢出去

    leader会维护一个与其基本保持同步的Replica列表，该列表称为ISR(in-sync Replica)，每个Partition都会有一个ISR，而且是由leader动态维护 ，如果一个follower比一个leader落后太多，或者超过一定时间未发起数据复制请求，则leader将其重ISR中移除 ，详细参考 kafka的高可用机制

7、kafka 为什么那么快

    1.Cache Filesystem Cache PageCache缓存。
    2.顺序写 由于现代的操作系统提供了预读和写技术，磁盘的顺序写大多数情况下比随机写内存还要快。
    3.Zero-copy 零拷⻉技术减少拷贝次数。
    4.Batching of Messages 批量量处理。合并小的请求，然后以流的方式进行交互，直顶网络上限。
    5.Pull 拉模式 使用拉模式进行消息的获取消费，与消费端处理能力相符。

8、kafka producer如何优化打入速度

    - 增加线程
    - 提高 batch.size
    - 增加更多 producer 实例
    - 增加 partition 数
    - 设置 acks=-1 时，如果延迟增大：可以增大 num.replica.fetchers（follower 同步数据的线程数）来调解
    - 跨数据中心的传输：增加 socket 缓冲区设置以及 OS tcp 缓冲区设置。

9、kafka producer 打数据，ack  为 0， 1， -1 的时候代表啥， 设置 -1 的时候，什么情况下，leader 会认为一条消息 commit了

    - 1（默认）  数据发送到Kafka后，经过leader成功接收消息的的确认，就算是发送成功了。在这种情况下，如果leader宕机了，则会丢失数据。
    - 0 生产者将数据发送出去就不管了，不去等待任何返回。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
    - 1 producer需要等待ISR中的所有follower都确认接收到数据后才算一次发送完成，可靠性最高。

    当ISR中所有Replica都向Leader发送ACK时，leader才commit，这时候producer才能认为一个请求中的消息都commit了。

10、kafka  unclean 配置代表啥，会对 spark streaming 消费有什么影响

    unclean.leader.election.enable 为true的话，意味着非ISR集合的broker 也可以参与选举，这样有可能就会丢数据，spark streaming在消费过程中拿到的 end offset 会突然变小，导致 spark streaming job挂掉。
    如果unclean.leader.election.enable参数设置为true，就有可能发生数据丢失和数据不一致的情况，Kafka的可靠性就会降低；而如果unclean.leader.election.enable参数设置为false，Kafka的可用性就会降低。

11、如果leader crash时，ISR为空怎么办

    kafka在Broker端提供了一个配置参数：unclean.leader.election,这个参数有两个值：
    true（默认）：允许不同步副本成为leader，由于不同步副本的消息较为滞后，此时成为leader，可能会出现消息不一致的情况。
    false：不允许不同步副本成为leader，此时如果发生ISR列表为空，会一直等待旧leader恢复，降低了可用性。

12、kafka的message格式是什么样的

    一个Kafka的Message由一个固定长度的header和一个变长的消息体body组成
    header部分由一个字节的magic(文件格式)和四个字节的CRC32(用于判断body消息体是否正常)构成。
    当magic的值为1的时候，会在magic和crc32之间多一个字节的数据：attributes(保存一些相关属性，
    比如是否压缩、压缩格式等等);如果magic的值为0，那么不存在attributes属性
    body是由N个字节构成的一个消息体，包含了具体的key/value消息


13、kafka中consumer group 是什么概念

    同样是逻辑上的概念，是Kafka实现单播和广播两种消息模型的手段。同一个topic的数据，会广播给不同的group；同一个group中的worker，只有一个worker能拿到这个数据。换句话说，对于同一个topic，每个group都可以拿到同样的所有数据，但是数据进入group后只能被其中的一个worker消费。group内的worker可以使用多线程或多进程来实现，也可以将进程分散在多台机器上，worker的数量通常不超过partition的数量，且二者最好保持整数倍关系，因为Kafka在设计时假定了一个partition只能被一个worker消费（同一group内）。


14、Kafka基本配置及性能优化

  ######14.1 硬件要求(Kafka 集群基本硬件的保证)
  <img alt="kafka-bb5bd82f.png" src="assets/kafka-bb5bd82f.png" width="" height="" >

  #######14.2 OS 调优
    - OS page cache：应当可以缓存所有活跃的 Segment（Kafka 中最基本的数据存储单位）；
    - fd 限制：100k+；
    - 禁用 swapping：简单来说，swap 作用是当内存的使用达到一个临界值时就会将内存中的数据移动到 swap 交换空间，但是此时，内存可能还有很多空余资源，swap 走的是磁盘 IO，对于内存读写很在意的系统，最好禁止使用 swap 分区;
    - TCP 调优；
    - JVM 配置
      - JDK 8 并且使用 G1 垃圾收集器；
      - 至少要分配 6-8 GB 的堆内存。

  ######14.3 Kafka 磁盘存储
    - 使用多块磁盘，并配置为 Kafka 专用的磁盘；
    - JBOD vs RAID10；
    - JBOD（Just a Bunch of Disks，简单来说它表示一个没有控制软件提供协调控制的磁盘集合，它将多个物理磁盘串联起来，提供一个巨大的逻辑磁盘，数据是按序存储，它的性能与单块磁盘类似）

    - JBOD 的一些缺陷：
      - 任何磁盘的损坏都会导致异常关闭，并且需要较长的时间恢复；
      - 数据不保证一致性；
      - 多级目录；
    - 社区也正在解决这么问题，可以关注 KIP 112、113：
      - 必要的工具用于管理 JBOD；
      - 自动化的分区管理；
      - 磁盘损坏时，Broker 可以将 replicas 迁移到好的磁盘上；
      - 在同一个 Broker 的磁盘间 reassign replicas；
    - RAID 10 的特点：
      - 可以允许单磁盘的损坏；
      - 性能和保护；
      - 不同磁盘间的负载均衡；
      - 高命中来减少 space；
      - 单一的 mount point；
    - 文件系统：
        - 使用 EXT 或 XFS；
        - SSD；

  ######14.4 基本的监控
    Kafka 集群需要监控的一些指标，这些指标反应了集群的健康度。

    - CPU 负载；
    - Network Metrics；
    - File Handle 使用；
    - 磁盘空间；
    - 磁盘 IO 性能；
    - GC 信息；
    - ZooKeeper 监控。
    - Kafka replica 相关配置及监控

  ######14.5 Kafka replica 相关配置及监控
    Kafka Replication:
    - Partition 有两种副本：Leader，Follower；
    - Leader 负责维护 in-sync-replicas(ISR)
      - replica.lag.time.max.ms：默认为10000，如果 follower 落后于 leader 的消息数超过这个数值时，leader 就将 follower 从 isr 列表中移除；
      - num.replica.fetchers，默认为1，用于从 leader 同步数据的 fetcher 线程数；
      - min.insync.replica：Producer 端使用来用于保证 Durability（持久性）；


  ######14.6 Under Replicated Partitions
    当发现 replica 的配置与集群的不同时，一般情况都是集群上的 replica 少于配置数时，可以从以下几个角度来排查问题：

    - JMX 监控项：
    kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions；

    - 可能的原因：
      - Broker 挂了？
      - Controller 的问题？
      - ZooKeeper 的问题？
      - Network 的问题？

    - 解决办法：
      - 调整 ISR 的设置；
      - Broker 扩容。

  ######14.7 Controller
    - 负责管理 partition 生命周期；
    - 避免 Controller’s ZK 会话超时：
      - ISR 抖动；
      - ZK Server 性能问题；
      - Broker 长时间的 GC；
      - 网络 IO 问题；
    - 监控：
      - kafka.controller:type=KafkaController,name=ActiveControllerCount，应该为1；
      - LeaderElectionRate。

  ######14.8 Unclean leader 选举

    允许不在 isr 中 replica 被选举为 leader。
      - 这是 Availability 和 Correctness 之间选择，Kafka 默认选择了可用性；
      - unclean.leader.election.enable：默认为 true，即允许不在 isr 中 replica 选为 leader，这个配置可以全局配置，也可以在 topic 级别配置；
      - 监控：kafka.controller:type=ControllerStats,name=UncleanLeaderElectionsPerSec。

  ######14.9 Broker 配置
    Broker 级别有几个比较重要的配置，一般需要根据实际情况进行相应配置的：

    - log.retention.{ms, minutes, hours} , log.retention.bytes：数据保存时间；
    - message.max.bytes, replica.fetch.max.bytes；
    - delete.topic.enable：默认为 false，是否允许通过 admin tool 来删除 topic；
    - unclean.leader.election.enable = false，参见上面；
    - min.insync.replicas = 2：当 Producer 的 acks 设置为 all 或 -1 时，min.insync.replicas 代表了必须进行确认的最小 replica 数，如果不够的话 Producer 将会报 NotEnoughReplicas 或 NotEnoughReplicasAfterAppend 异常；
    - replica.lag.time.max.ms（超过这个时间没有发送请求的话，follower 将从 isr 中移除）, num.replica.fetchers；
    - replica.fetch.response.max.bytes；
    - zookeeper.session.timeout.ms = 30s；
    - num.io.threads：默认为8，KafkaRequestHandlerPool 的大小。

15、Kafka 相关资源的评估
    集群评估:

    - Broker 评估
      - 每个 Broker 的 Partition 数不应该超过2k；
      - 控制 partition 大小（不要超过25GB）；
    - 集群评估（Broker 的数量根据以下条件配置）
      - 数据保留时间；
      - 集群的流量大小；
    - 集群扩容：
      - 磁盘使用率应该在 60% 以下；
      - 网络使用率应该在 75% 以下；
    - 集群监控
      - 保持负载均衡；
      - 确保 topic 的 partition 均匀分布在所有 Broker 上；
      - 确保集群的阶段没有耗尽磁盘或带宽。
      - Broker 监控

    Partition 数：

    - kafka.server:type=ReplicaManager,name=PartitionCount；
    - Leader 副本数：kafka.server:type=ReplicaManager,name=LeaderCount；
    - ISR 扩容/缩容率：kafka.server:type=ReplicaManager,name=IsrExpandsPerSec；
    - 读写速率：Message in rate/Byte in rate/Byte out rate；
    - 网络请求的平均空闲率：NetworkProcessorAvgIdlePercent；
    - 请求处理平均空闲率：RequestHandlerAvgIdlePercent。

    Topic 评估:

    - partition 数
      - Partition 数应该至少与最大 consumer group 中 consumer 线程数一致；
      - 对于使用频繁的 topic，应该设置更多的 partition；
      - 控制 partition 的大小（25GB 左右）；
      - 考虑应用未来的增长（可以使用一种机制进行自动扩容）；
    - 使用带 key 的 topic；
    - partition 扩容：当 partition 的数据量超过一个阈值时应该自动扩容（实际上还应该考虑网络流量）。


    合理地设置 partition

    - 根据吞吐量的要求设置 partition 数：
      - 假设 Producer 单 partition 的吞吐量为 P；
      - consumer 消费一个 partition 的吞吐量为 C；
      - 而要求的吞吐量为 T；
      - 那么 partition 数至少应该大于 T/P、T/c 的最大值；
    - 更多的 partition，意味着：
      - 更多的 fd；
      - 可能增加 Unavailability（可能会增加不可用的时间）；
      - client 端将会使用更多的内存。
      - 可能增加端到端的延迟；


    关于 Partition 的设置可以参考这篇文章 https://www.confluent.io/blog/how-choose-number-- topics-partitions-kafka-cluster ，这里简单讲述一下，Partition 的增加将会带来以下几个优点和缺点：

    - 增加吞吐量：对于 consumer 来说，一个 partition 只能被一个 consumer 线程所消费，适当增加 partition 数，可以增加 consumer 的并发，进而增加系统的吞吐量；
    - 需要更多的 fd：对于每一个 segment，在 broker 都会有一个对应的 index 和实际数据文件，而对于 Kafka Broker，它将会对于每个 segment 每个 index 和数据文件都会打开相应的 file handle（可以理解为 fd），因此，partition 越多，将会带来更多的 fd；
    - 可能会增加数据不可用性（主要是指增加不可用时间）：主要是指 broker 宕机的情况，越多的 partition 将会意味着越多的 partition 需要 leader 选举（leader 在宕机这台 broker 的 partition 需要重新选举），特别是如果刚好 controller 宕机，重新选举的 controller 将会首先读取所有 partition 的 metadata，然后才进行相应的 leader 选举，这将会带来更大不可用时间；
    - 可能增加 End-to-end 延迟：一条消息只有其被同步到 isr 的所有 broker 上后，才能被消费，partition 越多，不同节点之间同步就越多，这可能会带来毫秒级甚至数十毫秒级的延迟；
    - Client 将会需要更多的内存：Producer 和 Consumer 都会按照 partition 去缓存数据，每个 partition 都会带来数十 KB 的消耗，partition 越多, Client 将会占用更多的内存。


16、Producer 的相关配置、性能调优及监控

    Quotas

    - 避免被恶意 Client 攻击，保证 SLA；
    - 设置 produce 和 fetch 请求的字节速率阈值；
    - 可以应用在 user、client-id、或者 user 和 client-id groups；
    - Broker 端的 metrics 监控：throttle-rate、byte-rate；
    - replica.fetch.response.max.bytes：用于限制 replica 拉取请求的内存使用；
    - 进行数据迁移时限制贷款的使用，kafka-reassign-partitions.sh -- -throttle option。

    Kafka Producer

    - 使用 Java 版的 Client；
    - 使用 kafka-producer-perf-test.sh 测试你的环境；
    - 设置内存、CPU、batch 压缩；
      - batch.size：该值设置越大，吞吐越大，但延迟也会越大；
      - linger.ms：表示 batch 的超时时间，该值越大，吞吐越大、但延迟也会越大；
      - max.in.flight.requests.per.connection：默认为5，表示 client 在 blocking 之前向单个连接（broker）发送的未确认请求的最大数，超过1时，将会影响数据的顺序性；
      - compression.type：压缩设置，会提高吞吐量；
      - acks：数据 durability 的设置；
    - 避免大消息
      - 会使用更多的内存；
      - 降低 Broker 的处理速度；


    性能调优

    - 如果吞吐量小于网络带宽
      - 增加线程；
      - 提高 batch.size；
      - 增加更多 producer 实例；
      - 增加 partition 数；
    - 设置 acks=-1 时，如果延迟增大：可以增大 num.replica.fetchers（follower 同步数据的线程数）来调解；
    - 跨数据中心的传输：增加 socket 缓冲区设置以及 OS tcp 缓冲区设置。


    Prodcuer 监控

    - batch-size-avg
    - compression-rate-avg
    - waiting-threads
    - buffer-available-bytes
    - record-queue-time-max
    - record-send-rate
    - records-per-request-avg


17、Kafka Consumer 配置、性能调优及监控
    Kafka Consumer

    - 使用 kafka-consumer-perf-test.sh 测试环境；
    - 吞吐量问题：
      - partition 数太少；
      - OS page cache：分配足够的内存来缓存数据；
      - 应用的处理逻辑；
    - offset topic（__consumer_offsets）
    - offsets.topic.replication.factor：默认为3；
    - offsets.retention.minutes：默认为1440，即 1day； MonitorISR，topicsize；
    - offset commit较慢：异步 commit 或 手动 commit。


    Consumer 配置

    - fetch.min.bytes 、fetch.max.wait.ms；
    - max.poll.interval.ms：调用 poll() 之后延迟的最大时间，超过这个时间没有调用 poll() 的话，就会认为这个 consumer 挂掉了，将会进行 rebalance；
    - max.poll.records：当调用 poll() 之后返回最大的 record 数，默认为500；
    - session.timeout.ms；
    - Consumer Rebalance
      - check timeouts
      - check processing times/logic
      - GC Issues
    - 网络配置；

    Consumer 监控
    consumer 是否跟得上数据的发送速度。

    - Consumer Lag：consumer offset 与 the end of log（partition 可以消费的最大 offset） 的差值；
    - 监控
      - metric 监控：records-lag-max；
      - 通过 bin/kafka-consumer-groups.sh 查看；
      - 用于 consumer 监控的 LinkedIn’s Burrow；
    - 减少 Lag
      - 分析 consumer：是 GC 问题还是 Consumer hang 住了；
      - 增加 Consumer 的线程；
      - 增加分区数和 consumer 线程；


18、如何保证数据不丢

  这个是常用的配置:
  <img alt="kafka-1244f695.png" src="assets/kafka-1244f695.png" width="" height="" >

    block.on.buffer.full：默认设置为 false，当达到内存设置时，可能通过 block 停止接受新的 record 或者抛出一些错误，默认情况下，Producer 将不会抛出 BufferExhaustException，而是当达到 max.block.ms 这个时间后直接抛出 TimeoutException。
    设置为 true 的意义就是将 max.block.ms 设置为 Long.MAX_VALUE，未来版本中这个设置将被遗弃，推荐设置 max.block.ms。

![](assets/markdown-img-paste-20190328205347632.png)
