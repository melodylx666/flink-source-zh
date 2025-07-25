### 两阶段提交(2PC)

两阶段提交(2PC)是最基础的分布式一致性协议，Flink主要用它来实现exactly once，完成事务性写入。

> 这里的exactly once是端到端的，而Flink内部的exactly once是通过checkpoint实现的。

首先，我们必须知道，在分布式环境中，为了让每个节点都能感知其它节点上事务的执行情况，需要引入一个中心节点来统一协调处理所有节点的执行逻辑，这
个中心节点就叫做协调者(Coordinator)，而其它被中心节点协调的节点就是参与者。

而两阶段提交，故名思义，就是将分布式事务划分成了两个阶段，分别是提交请求(表决)阶段和提交(执行正常或异常)阶段。其中，协调者会根据参与者对第一
阶段，也就是提交请求/表决阶段的相应来决定是否真正的执行事务，也就是第二步执行提交/执行正常阶段或是执行提交/执行异常阶段。

下面分别对这两个阶段进行介绍。

第一阶段，提交请求(表决)阶段：
  * 协调者向所有参与者发送prepare请求与事务内容，询问是否可以准备事务的提交，并等待参与者的响应;

  * 参与者执行事务中包含的操作，并记录用于回滚的undo逻辑日志和用于重放的redo物理日志，但是并不真正的执行提交;

  * 参与者向协调者返回事务操作的执行结果，执行成功返回yes，否则返回no;

第二阶段，提交(执行)阶段分为正常和异常两种情况：
  * 如果所有的参与者都返回yes表示事务可以被提交：协调者向所有的参与者发送提交请求，参与者收到协调者发送的提交请求后，真正的进行事务的提交操作，
  并释放其占用的事务资源，向协调者返回确认信息，协调者收到所有的参与者返回的确认信息，分布式事务成功完成;

  * 如果有参与者返回no或者是超时未返回，说明事务中断，需要回滚：协调者向所有参与者发送回滚请求，参与者收到回滚请求后，根据undo日志回滚到事务执行
  前的状态，释放占用的事务资源，并向协调者返回ack，协调者收到所有参与者的ack信息，事务回滚完成;

从上面的介绍可以看出，2PC的实现非常简单，当然也存在一定的缺陷：
  * 协调者存在单点故障，如果协调者宕机，则整个2PC逻辑就完全不能运行;

  * 整个执行过程完全同步，参与者在等待其它参与者响应时都处于阻塞状态，高并发场景下存在性能问题;

  * 依然可能存在不一致的风险，比如如果在第二阶段由于网络或其它原因，只有部分参与者收到了commit请求，此时就会造成部分参与者进行了事务的提交而其它
  参与者未提交的情况;

  * 如果有节点无法响应，只能靠协调者自身的超时机制来决定是中断事务。

Flink作为一款时下最火爆的流式处理引擎，能够提供exactly once的语义保证，有如下两层含义：
1. 端到端的exactly once语义保证。即输入、处理程序、输出这个三个部分协同作用，共同实现整个流程的精确一次语义。这里主要指的是Producer，也就是比如FlinkToKafka，因为它的下游数据源是外部系统，而不是Flink内部组件。
2. Flink内部的excatly once语义保证。Flink内部是通过检查点机制(checkpoint)和分布式快照来实现exactly once的。同时，在Flink中提供了基于2PC的SinkFunction，叫做TwoPhaseCommitSinkFunction来对输入端的事务性写入提供基础性的支持，这是个抽象类，所
有需要保证exactly once的Sink逻辑都需要继承这个类。

TwoPhaseCommitSinkFunction类中有四个抽象方法与2pc的过程相关，分别是：
  * beginTransaction():开始一个事务，返回事务信息的句柄;

  * preCommit():预提交阶段(提交请求)的逻辑;

  * commit():正式提交阶段的逻辑;

  * abort():取消事务;

由此可见，输出端也必须能对事务性写入提供支持，当然如果输出sink也是kafka 0.11及以上版本肯定是没问题的，如果是其它的输出端也需要其支持事务或实现了写入的幂等才行。以kafka来看，在FlinkKafkaProducer011类中实现了beginTransaction()方法。当要求支持exactly once语义时，每次都会
调用createTransactionalProducer()来生成包含事务ID的producer。而preCommit()方法的实现就很简单了，就是直接调用了producer的flush()方法，它是在Sink算子进行snapshot操作时被调用的。

以 FlinkKafkaProducer011 为例，该类实现了 TwoPhaseCommitSinkFunction 接口，支持 Kafka 0.11+ 的事务性写入。

1. beginTransaction()：创建一个事务性的 Kafka Producer，并分配事务 ID。

2. preCommit()：调用 KafkaProducer.flush()，将缓冲的数据写入磁盘，但不提交事务。

3. commit()：调用 KafkaProducer.commitTransaction()，正式提交事务。

4. abort()
调用 KafkaProducer.abortTransaction()，回滚事务。

具体流程如下：

在两次快照的间隔期间，Sink算子将数据发送到数据汇引擎中，但是数据汇引擎会先将数据视为uncommitted，此时这些数据不能被下游读取到。等到Flink快照完毕，Sink收到JM的快照完成信息，就可以进行commit()了。

1. Checkpoint Barrier 插入
JobManager 在数据流中插入一个 barrier，触发所有算子的 checkpoint。

2. 预提交阶段（prepare）
当 barrier 到达 Kafka Sink 时，触发 preCommit()，调用 flush() 刷新数据到 Kafka，但不提交。

3. 提交阶段（commit）
当所有 checkpoint 成功完成后，调用 commit()，Kafka Producer 提交事务，确保所有数据写入成功。

4. 失败回滚
如果 checkpoint 失败，调用 abort() 回滚事务，恢复到之前的状态。

