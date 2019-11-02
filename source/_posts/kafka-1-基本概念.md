---
title: 深入理解kafka读书笔记
categories:
  - 中间件
tags:
  - mq
  - kafka
abbrlink: 46459
---

### kafka 读书笔记

#### 一.术语

​    ![123](<https://raw.githubusercontent.com/mzl1989325/ImageBed/master/mq/kafka/kafka.jpg>)

* Producer:生产者，也就是发送消息的一方。生产者负责创建消息，然后将其投递到Kafka中

* Consumer:消费者,也就是接收消息的一方。消费者连接到Kafka上并接收消息，进而进行相应的业务处理。

* Broker:服务代理节点。对于kafka而言，Broker可以简单的看做一个独立的kafka服务节点或Kafka服务实例。大多数情况下可以将Broker看作一台Kafka服务器，前提是这台服务器上只部署了一个Kafka实例。一个或多个Broker组成了一个Kafka集群。

* 在Kafka中还有两个重要的概念——主题（Topic）与分区(Partition)。Kafka中的消息以主题为单位进行归类，生产者负责将消息发送到特定的主题，而消费者负责订阅主题进行消费。

  * 主题是一个逻辑上的概念，他可以细分多个分区，**一个分区只属于单个主题**。

  * **同一主题下的不同分区包含的消息是不同的**.分区在存储层面可以看做一个可追加的日志文件，消息被分区追加到分区日志文件的时候会分配一个特定的偏移量（offset）。offset是消息在分区中的唯一标识，Kafka通过它来保证消息在分区内的顺序性，不过offset不跨越分区，也就是说Kafka保证分区有序而不是主题有序。

  * 如图所示，主题中有4个分区，消息被顺序追加到每个日志尾部。Kafka中的分区可以分布在不同的服务器上(broker),也就是说一个主题可以横跨多个broker.以此来提供比单个broker更强大的性能。

     ![123](<https://raw.githubusercontent.com/mzl1989325/ImageBed/master/mq/kafka/message-write.jpg>)

    每一条消息被发送到 broker 之前，会根据分区规则选择存储到哪个具体的分区 。 如果分区规则设定得合理，所有的消息都可以均匀地分配到不同的分区中 。 如果一个主题只对应一个文件，那么这个文件所在的机器 I/O 将会成为这个主题的性能瓶颈，而分区解决了这个问题 。 在创建主题的时候可以通过指定的参数来设置分区的个数，当然也可以在主题创建完成之后去修改分区的数量，通过增加分区的数量可以实现水平扩展。 

    Kafka 为分区引入了多副本 （ Replica ） 机制， 通过增加副本数量可以提升容灾能力。同一分区的不同副本中保存的是相同的消息（在同一时刻，副本之间并非完全一样），副本之间是“ 一主多 从”的关系，其中 leader 副本负 责处理读写请求 ， follower 副本只负 责与 lead er 副本的消息同步。副本处于不同的 broker 中 ，当 leader 副本出现故障时，从 follower 副本中重新选举新的 leader 副本对外提供服务。 **Kafka 通过多副本机制实现了故障的自动转移，当 Kafka 集群中某个 broker 失效时仍然能保证服务可用** 。

    如图 3 所示， Kafka 集群中有 4 个 broker，某个主题中有 3 个分区，且副本因子（即副本个数〉也为 3 ，如此每个分区便有 l 个 leader 副本和 2 个 follower 副本。生产者和消费者只与 leader副本进行交互，而 follower 副本只负责消息的同步，很多时候 follower 副本中的消息相对 leader副本而言会有一定的滞后。 

    ![123](https://raw.githubusercontent.com/mzl1989325/ImageBed/master/mq/kafka/multi.jpg)

    Kafka 消费端也具备一 定 的 容灾能力。 Consumer 使用拉 （ Pull ）模式从服务端拉取消息，并且保存消费 的具体位置 ， 当消费者看机后恢复上线时可以根据之前保存的消费位置重新拉取需要的消息进行消 费 ，这样就不会造成消息丢失 。

    分区中 的所有副本统称为 AR ( Assigned Replicas ） 。 **所有与 leader 副本保持一定程度同步的副本（包括 leader 副本在内〕组成 ISR （In-Sync Replicas )** , ISR 集合是 AR 集合中 的一个子集 。 消息会先发送到leader 副本，然后 follower 副本才能从 leader 副本中拉取消息进行同步，同步期间内 follower 副本相对于leader 副本而言会有一定程度的滞后 。 前面所说的“一定程度的同步”是指可忍受的滞后范围，这个范围可以通过参数进行配置 。 **与 leader 副本同步滞后过多的副本（不包括 leader 副本）组成 OSR ( Out-of-Sync Replicas ）**，由此可见， AR=ISR+OSR 。在正常情况下， 所有的 follower 副本都应该与 leader 副本保持一定程度 的同步，即 AR=ISR,OSR 集合为空。 

    leader 副本负 责维护和跟踪 ISR 集合中所有 follower 副 本 的滞后状态， 当 follower 副本落后太多或失效时， leader 副本会把它从 ISR 集合中剔除 。 如果 OSR 集合中有 follower 副本 “追上’了 leader 副本，那么 leader 副本会把它从 OSR 集合转移至 ISR 集合 。 默认情况下， 当 leader 副本发生故障时，只 有在 ISR 集合中的副本才有资格被选举为新的 leader， 而在 OSR 集合中的副本则没有任何机会（不过这个原则也可以通过修改相应的参数配置来改变）。

    ISR 与 HW 和 LEO 也有紧密的关系 。 HW 是 High Watermark 的缩写，俗称高水位，它标识了 一个特定 的消息偏移量（ offset ），**消费者只能拉取到这个 offset 之前的消息** 。 

    如图 4 所示，它代表一个日志文件，这个日志文件中有 9 条消息，第一条消息的 offset (LogStartOffset)为0，最后一条的offset为8，offset为9的消息用虚线框标识，代表下一条待写入的消息。日志文件的HW为6，标识消费者只能拉取到offset在0到5之间的消息，而offset为6的消息对消费者而言是不可见的。 

    

    ![123](<https://raw.githubusercontent.com/mzl1989325/ImageBed/master/mq/kafka/offset.jpg>) 

  ​                                                                       图4 分区中各种偏移量说明

  

  ​      LEO是LogEndOffset的缩写，它标识当前日志文件中下一条待写入消息的offset,图4中offset为9的位置即为当前日志文件的LEO，LEO的大小相当于当前日志分区中最后一条消息的offset值加1.分区ISR集合中的每个副本都会委会自身的LEO，而ISR集合中最小的LEO即为分区的HW,对于消费者而言只能消费HW之前的消息。

  ​     如下图所示，加入某个分区的ISR集合找那个有3个副本，即一个leader副本和2个follower副本，此时分区的LEO和HW都为3。消息3和消息4从生产者发出之后会被先存入leader副本，如图6所示。

  ![123](https://raw.githubusercontent.com/mzl1989325/ImageBed/master/mq/kafka/wirte-1.jpg)

   ![123](<https://raw.githubusercontent.com/mzl1989325/ImageBed/master/mq/kafka/write-2.jpg>)

​	在消息写入leader副本之后，follower副本会发送拉去请求来拉去消息3和消息4以进行消息同				        步。

​       在同步过程中，不同的 follower 副本的同步效率也不尽相同。如图 7 所示， 在某一时刻
follower1 完全跟上了 leader 副本而 follower2 只同步了消息 3 ，如此 leader 副本的 LEO 为 5,
follower1 的 LEO 为 5 , follower2 的 LEO 为 4 ， 那么当前分区的 HW 取最小值 4 ，此时消费者可
以消 费到 offset 为 0 至 3 之间的消息。  

![123](<https://raw.githubusercontent.com/mzl1989325/ImageBed/master/mq/kafka/write-3.jpg>)

​	如图8所示，所有的副本都成功写入了消息3和消息4，整个分区的HW和LEO都变为5，因此消费者可以消费到offset为4的消息了。

![123](<https://raw.githubusercontent.com/mzl1989325/ImageBed/master/mq/kafka/write-4.jpg>)

由此可见， Kafka 的复制机制既不是完全的同步复制，也不是单纯的异步复制。事实上，
同步复制要求所有能工作的 follower 副本都复制完，这条消息才会被确认为已成功提交，这种
复制方式极大地影响了性能。而在异步复制方式下， follower 副本异步地从 leader 副本中 复制数
据，数据只要被 leader 副本写入就被认为已经成功提交。在这种情况下，如果 follower 副本都
还没有复制完而落后于 leader 副本，突然 leader 副本着机，则会造成数据丢失。 Kafka 使用的这
种 ISR 的方式则有效地权衡了数据可靠性和性能之间的关系。 