

> 注意：先去了解消息队列：https://www.zhihu.com/question/34243607


## 为什么选择RocketMQ作消息队列
- ActiveMQ 的社区算是比较成熟，但是较目前来说，**ActiveMQ 的性能比较差，而且版本迭代很慢**，不推荐使用。
- RabbitMQ 在**吞吐量方面虽然稍逊于 Kafka 和 RocketMQ** ，但是由于它基于 erlang 开发，所以并发能力很强，性能极其好，延时很低，达到微秒级。但是也因为 RabbitMQ 基于 erlang 开发，所以国内很少有公司有实力做erlang源码级别的研究和定制。如果**业务场景对并发量要求不是太高（十万级、百万级），那这四种消息队列中，RabbitMQ 一定是你的首选**。如果是大数据领域的实时计算、日志采集等场景，用 Kafka 是业内标准的，绝对没问题，社区活跃度很高，绝对不会黄，何况几乎是全世界这个领域的事实性规范。
- RocketMQ 阿里出品，**Java 系开源项目**，源代码我们可以直接阅读，然后可以**定制自己公司的MQ**，并且 RocketMQ 有阿里巴巴的实际业务场景的实战考验。其次具有分布式事务消息的功能，可以达到消息的最终一致性。
- kafka 的特点其实很明显，就是仅仅提供较少的核心功能，但是提供超高的吞吐量，ms 级的延迟，极高的可用性以及可靠性，而且分布式可以任意扩展。同时 kafka 最好是支撑较少的 topic 数量即可，保证其超高吞吐量。**kafka 唯一的一点劣势是有可能消息重复消费**，那么对数据准确性会造成极其轻微的影响，在大数据领域中以及日志采集中，这点轻微影响可以忽略这个特性天然适合大数据实时计算以及日志收集。

## RocketMQ组件
- Producer:消息发布的角色，主要负责把消息发送到Broker，支持分布式集群方式部署。
- Consumer:消息消费者的角色，主要负责从Broker订阅消息消费，支持分布式集群方式部署。
- Broker:消息存储的角色，主要负责消息的存储、投递和查询，以及服务高可用的保证，支持分布式集群方式部署。
- NameServer:是一个非常简单的Topic路由注册中心，其角色类似于Dubbo中依赖的Zookeeper，支持Broker动态注册和发现。
    - 服务注册：NameServer接收Broker集群注册的信息，保存下来作为路由信息的基本数据，并提供心跳检测机制，检查Broker是否存活。
    - 路由信息管理：NameServer保存了Broker集群的路由信息，用于提供给客户端查询Broker的队列信息。Producer和Consumer通过NameServer可以知道Broker集群的路由信息，从而进行消息的投递和消费。

## MQ在高并发情况下，假设队列满了如何防止消息丢失？
- 生产者可以采用重试机制。因为消费者会不停的消费消息，可以重试将消息放入队列。
- 死信队列，可以理解为备胎(推荐)
   - 即在消息过期，队列满了，消息被拒绝的时候，都可以扔给死信队列。
   - 如果出现死信队列和普通队列都满的情况，此时考虑消费者消费能力不足，可以对消费者开多线程进行处理。

## RocketMQ事务消息原理

![mq事务消息-NaWGxH](https://gitee.com/dreamcater/blog-img/raw/master/uPic/mq事务消息-NaWGxH.png)

想了想，mq也可以
1. 先给Brock发送一条消息：我要下单了，注意哈
2. Brock给本地事务回馈消息：ack，好的，我知道了（半投递状态，消费端看不到）
3. 本地事务开始执行业务逻辑，这里首先（校验场次id的座位是否重复，如果没有，直接执行下单：这两个业务非原子，上个锁，要不然可能会出现同样的座位）。下单成功，则返回commit给Brock。消费者此时就可以看到这条消息了。
4. 如果下单不成功，则返回rollback，Brock一般三天自动删除该无效的消息，消费者也看不到。
5. 消费者看到了这条消息，调用绑定座位服务，如果失败了，则重试。（消费端不能失败，要不然不能保持一致，如果还是一直失败，则人工处理。） 注意：幂等性

## 谈谈死信队列
**死信队列用于处理无法被正常消费的消息，即死信消息**。

当一条消息初次消费失败，**消息队列 RocketMQ 会自动进行消息重试**；达到最大重试次数后，若消费依然失败，则表明消费者在正常情况下无法正确地消费该消息，此时，消息队列 RocketMQ 不会立刻将消息丢弃，而是将其发送到该**消费者对应的特殊队列中**，该特殊队列称为**死信队列**。

**死信消息的特点**：

- 不会再被消费者正常消费。
- 有效期与正常消息相同，均为 3 天，3 天后会被自动删除。因此，请在死信消息产生后的 3 天内及时处理。

**死信队列的特点**：

- 一个死信队列对应一个 Group ID， 而不是对应单个消费者实例。
- 如果一个 Group ID 未产生死信消息，消息队列 RocketMQ 不会为其创建相应的死信队列。
- 一个死信队列包含了对应 Group ID 产生的所有死信消息，不论该消息属于哪个 Topic。

消息队列 RocketMQ 控制台提供对死信消息的查询、导出和重发的功能。

## 使用异步消息时如何保证数据的一致性

- **借助数据库的事务**：这需要在数据库中创建一个**本地消息表**，这样可以通过**一个事务来控制本地业务逻辑更新**和**本地消息表的写入在同一个事务中**，一旦消息落库失败，则直接全部回滚。如果消息落库成功，后续就可以根据情况基于本地数据库中的消息数据对消息进行重投了。关于本地消息表和消息队列中状态如何保持一致，可以采用 2PC 的方式。在发消息之前落库，然后发消息，在得到同步结果或者消息回调的时候更新本地数据库表中消息状态。然后只需要通过**定时轮询**的方式对状态未已记录但是未发送的消息重新投递就行了。但是这种方案有个前提，就是要求消息的消费者**做好幂等控制**，这个其实异步消息的消费者一般都需要考虑的。
- 除了使用数据库以外，还可以使用 **Redis** 等缓存。这样就是无法利用关系型数据库自带的事务回滚了。

## RockMQ不适用Zookeeper作为注册中心的原因，以及自制的NameServer优缺点？
- ZooKeeper 作为支持**顺序一致性**的中间件，在某些情况下，它为了满足一致性，会丢失一定时间内的**可用性**，RocketMQ 需要注册中心只是为了**发现组件地址**，在某些情况下，RocketMQ 的注册中心可以出现数据不一致性，这同时也是 **NameServer 的缺点，因为 NameServer 集群间互不通信，它们之间的注册信息可能会不一致**。
- 另外，当有新的服务器加入时，**NameServer 并不会立马通知到 Produer**，而是由 **Produer 定时去请求 NameServer 获取最新的 Broker/Consumer 信息**（这种情况是通过 Producer 发送消息时，负载均衡解决）
- 包括组件通信间使用 Netty 的自定义协议
- 消息重试负载均衡策略（具体参考 Dubbo 负载均衡策略）
- 消息过滤器（Producer 发送消息到 Broker，Broker 存储消息信息，Consumer 消费时请求 Broker 端从磁盘文件查询消息文件时,在 Broker 端就使用过滤服务器进行过滤）
- Broker 同步双写和异步双写中 Master 和 Slave 的交互

## 实现延迟队列
rocketmq发送延时消息时先把消息按照延迟时间段发送到指定的队列中(rocketmq把每种延迟时间段的消息都存放到同一个队列中)然后通过一个定时器进行轮训这些队列，查看消息是否到期，如果到期就把这个消息发送到指定topic的队列中，这样的好处是同一队列中的消息延时时间是一致的，还有一个好处是这个队列中的消息时按照消息到期时间进行递增排序的，说的简单直白就是队列中消息越靠前的到期时间越早

缺点：定时器采用了timer，timer是单线程运行，如果延迟消息数量很大的情况下，可能单线程处理不过来，造成消息到期后也没有发送出去的情况

改进点：可以在每个延迟队列上各采用一个timer，或者使用timer进行扫描，加一个线程池对消息进行处理，这样可以提供效率

## 消息队列如何保证顺序消费
生产者中把 orderId 进行取模，把相同模的数据放到 messagequeue 里面，消费者消费同一个 messagequeue，只要消费者这边有序消费，那么可以保证数据被顺序消费。

RocketMQ：顺序消息一般使用集群模式，是指对消息消费者内的线程池中的线程对消息消费队列只能串行消费。并并发消息消费最本质的区别是消息消费时必须成功锁定消息消费队列，在Broker端会存储消息消费队列的锁占用情况。
[https://blog.csdn.net/AAA821/article/details/86650471](https://blog.csdn.net/AAA821/article/details/86650471)

## RocketMQ消息失败策略
[https://developer.aliyun.com/article/717340](https://developer.aliyun.com/article/717340)

## RocketMQ顺序消息消费
[https://juejin.im/post/6844903982805024782](https://juejin.im/post/6844903982805024782)

[https://www.cnblogs.com/hzmark/p/orderly_message.html](https://www.cnblogs.com/hzmark/p/orderly_message.html)

## 零拷贝
都知道内核和用户态了，不必多说了
1. 从磁盘复制数据到内核态内存；
2. 从内核态内存复制到用户态内存；
3. 然后从用户态内存复制到网络驱动的内核态内存；
4. 最后是从网络驱动的内核态内存复制到网卡中进行传输。

但，通过使用mmap的方式，可以省去向用户态的内存复制，提高速度。这种机制在Java中是通过MappedByteBuffer实现的(零拷贝)

[https://zhuanlan.zhihu.com/p/83398714](https://zhuanlan.zhihu.com/p/83398714)

## 消息存储
### 存储过程
1. 消息生产者发送消息
2. MQ收到消息，将消息进行持久化，在存储中新增一条记录
3. 返回ACK给生产者
4. MQ push 消息给对应的消费者，然后等待消费者返回ACK
5. 如果消息消费者在指定时间内成功返回ack，那么MQ认为消息消费成功，在存储中删除消息，即执行第6步；如果MQ在指定时间内没有收到ACK，则认为消息消费失败，会尝试重新push消息,重复执行4、5、6步骤
6. MQ删除消息

想说一点，activeMQ的存储介质DB，这就影响了存储效率，其他几位MQ采用的文件系统，并且依照顺序写，极大跟随了SSD的步伐。


### 存储结构
RocketMQ消息的存储是由ConsumeQueue和CommitLog配合完成的，消息真正的物理存储文件是CommitLog，ConsumeQueue是消息的逻辑队列，类似数据库的索引文件，存储的是指向物理存储的地址。每个Topic下的每个Message Queue都有一个对应的ConsumeQueue文件。

- CommitLog：存储消息的元数据
- ConsumerQueue：存储消息在CommitLog的索引
- IndexFile：为了消息查询提供了一种通过key或时间区间来查询消息的方法，这种通过IndexFile来查找消息的方法不影响发送与消费消息的主流程

## 刷盘机制
**同步机制**：在返回写成功状态时，消息已经被写入磁盘。具体流程是，消息写入内存的PAGECACHE后，立刻通知刷盘线程刷盘， 然后等待刷盘完成，刷盘线程执行完成后唤醒等待的线程，返回消息写成功的状态。
1. 封装刷盘请求
2. 提交刷盘请求
3. 线程阻塞5秒，等待刷盘结束

服务那边：
1. 加锁
2. 遍历requestsRead
3. 刷盘
4. 唤醒发送消息客户端
5. 更新刷盘监测点

**异步机制**：在返回写成功状态时，消息**可能**只是被写入了内存的PAGECACHE，写操作的返回快，吞吐量大；当内存里的消息量积累到一定程度时，统一触发写磁盘动作，快速写入。

在消息追加到内存后，立即返回给消息发送端。如果开启transientStorePoolEnable，RocketMQ会单独申请一个与目标物理文件（commitLog）同样大小的堆外内存，该堆外内存将使用内存锁定，确保不会被置换到虚拟内存中去，消息首先追加到堆外内存，然后提交到物理文件的内存映射中，然后刷写到磁盘。如果未开启transientStorePoolEnable，消息直接追加到物理文件直接映射文件中，然后刷写到磁盘中。

开启transientStorePoolEnable后异步刷盘步骤:

1. 将消息直接追加到ByteBuffer（堆外内存）
2. CommitRealTimeService线程每隔200ms将ByteBuffer新追加内容提交到MappedByteBuffer中
3. MappedByteBuffer在内存中追加提交的内容，wrotePosition指针向后移动
4. commit操作成功返回，将committedPosition位置恢复
5. FlushRealTimeService线程默认每500ms将MappedByteBuffer中新追加的内存刷写到磁盘

## 路由管理
### 心跳机制
- RocketMQ路由注册是通过Broker与NameServer的心跳功能实现的。
- Broker启动时向集群中所有的NameServer发送心跳信息，每隔30s向集群中所有NameServer发送心跳包，NameServer收到心跳包时会更新brokerLiveTable缓存中BrokerLiveInfo的lastUpdataTimeStamp信息，然后NameServer每隔10s扫描brokerLiveTable，如果连续120S没有收到心跳包，NameServer将移除Broker的路由信息同时关闭Socket连接。

### 删除路由
- `Broker`每隔30s向`NameServer`发送一个心跳包，心跳包包含`BrokerId`，`Broker`地址，`Broker`名称，`Broker`所属集群名称、`Broker`关联的`FilterServer`列表。但是如果`Broker`宕机，`NameServer`无法收到心跳包，此时`NameServer`如何来剔除这些失效的`Broker`呢？
- `NameServer`会每隔10s扫描`brokerLiveTable`状态表，如果`BrokerLive`的**lastUpdateTimestamp**的时间戳距当前时间超过120s，则认为`Broker`失效，移除该`Broker`，关闭与`Broker`连接，同时更新`topicQueueTable`、`brokerAddrTable`、`brokerLiveTable`、`filterServerTable`。

### 路由发现
RocketMQ路由发现是非实时的，当Topic路由出现变化后，NameServer不会主动推送给客户端，而是由客户端定时拉取主题最新的路由。

其实就是一张图
![](https://imgkr.cn-bj.ufileos.com/6ac077ee-bc03-4785-9200-426457563858.png)




## 长轮训

拉取长轮询分析：源码上实际上还是个监听器
- RocketMQ未真正实现消息推模式，而是消费者主动向消息服务器拉取消息，RocketMQ推模式是循环向消息服务端发起消息拉取请求，如果消息消费者向RocketMQ拉取消息时，消息未到达消费队列时，如果不启用长轮询机制，则会在服务端等待shortPollingTimeMills时间后（挂起）再去判断消息是否已经到达指定消息队列，如果消息仍未到达则提示拉取消息客户端PULL—NOT—FOUND（消息不存在）；
- 如果开启长轮询模式，RocketMQ一方面会每隔5s轮询检查一次消息是否可达，同时一有消息达到后立马通知挂起线程再次验证消息是否是自己感兴趣的消息，如果是则从CommitLog文件中提取消息返回给消息拉取客户端，否则直到挂起超时，超时时间由消息拉取方在消息拉取是封装在请求参数中，PUSH模式为15s，PULL模式通过DefaultMQPullConsumer#setBrokerSuspendMaxTimeMillis设置。RocketMQ通过在Broker客户端配置longPollingEnable为true来开启长轮询模式。


说白了， push模式只不过在pull模式下加了个监控......哈哈哈哈