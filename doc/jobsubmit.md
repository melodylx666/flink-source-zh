### 任务提交

数据中台的目的(或者说所有中台的目的)都是让数据(或其它资源)持续的使用起来，通过中台提供的工具、方法和运行机制，将数据(或其它)资源变成一种能力，让数据(或其它资源)能更方便的为业务所使用。

回到Flink，它实际上是Google Dataflow模型的一种实现，其设计与Dataflow模型高度贴合，感兴趣的话可以去研究一下Dataflow，对于理解Flink的设计非常有帮助。目前，Flink为了全面对标Spark从而构建属于自己的生态，加上正在和阿里内部使用的Flink版本也就是Blink进行合并，版本正在进行着快速的迭代升级，这在给我们不断带来新功能新特性的同时，也给使用它和分析其源码带来了不小的挑战。

但是不管它的组件和代码怎么变，其基本组件模型不会变，任务的提交不会变，所以这里也就先从基本组件和任务的提交开始分析。Flink中的组件图及其交互如下图：
![Flink组件](../images/flinkcomponent.png "Flink组件")

需要注意的是，Flink中采用的是单进程多线程的执行方式(这与Spark的多进程的执行方式不太一样)，因此，当一个TaskManager上有多个Slot时，多个任务会共享同一个JVM的资源，比如TCP连接、心跳信息，甚至共享数据集、数据结构，这会降低每个任务的吞吐率。

另外，如果某个task在运行时将整个work的内存占满，这个异常的任务可能会把整个TM进程kill掉，这样运行在之上的其他任务也都被kill掉了。当一个TaskManager上只有一个Slot时，每个任务就会在单独的JVM里执行，可达到应用程序独立部署、资源隔离的目的，防止异常的应用干扰其他无关的应用。因此，在Flink 1.7以后的版本中，一个TaskManager默认只会有一个Slot，如果需要设置多个需要手动修改该配置，从而达到任务间资源隔离的目的。

这里可以通过一个具体的例子来对比上述Flink和Spark的不同。

假设有一个worker节点。

- 对于Flink来说，一个Worker节点上运行一个TaskManager JVM进程，所有分发到本节点的subtask在TaskManager进程中通过线程并发运行。而Slot是一个TaskManager进程的资源的固定子集(只是托管内存均分并隔离，但是CPU不会隔离)，其数量等于TaskManager并发处理task的数量。如果一个TaskManager只有一个Slot，则每个task都运行在单独的JVM中，则完全隔离。而如果一个TaskManager有多个Slot，则多个task运行在同一个JVM中，共享资源，比如连接，数据集等。
- 对于Spark来说，一个worker节点上运行一个worker进程，以及多个ExecutorBackend进程(彼此之间相互隔离，每个Executor都可以并发运行多个task)。具体数量是通过单个节点资源/单个executor资源来计算的。

再来看一下Flink任务在被提交到Yarn上后会经过的处理流程(Flink job有session，Job，application三种执行模式，这里将其看做application模式),具体如下:

 ![Flink提交到yarn](../images/flinksubmittoyarn.png "Flink提交到yarn")

 1. Client从客户端代码生成的StreamGraph提取出JobGraph;

 2. 上传JobGraph和对应的jar包;

 3. 启动App Master;

 4. 启动JobManager;

 5. 启动ResourceManager;

 6. JobManager向ResourceManager申请slots;

 7. ResourceManager向Yarn ResourceManager申请Container;

 8. 启动申请到的Container;

 9. 启动的Container作为TaskManager向ResourceManager注册;

 10. ResourceManger向TaskManager请求slot;

 11. TaskManager提供slot给JobManager,让其分配任务执行.

 上面的流程主要包含Client,JobManager,ResourceManager,TaskManager共四个部分.接下来就对每个部分进行详细的分析.

### 生成StreamGraph

在用户编写一个Flink任务之后是怎么样一步步转换成Flink的第一层抽象StreamGraph的呢?本节将会对此进行详细的介绍.

StreamGraph生成的主要流程如下:

1. 将用户编写到map，keyby等操作包装为Transformation,添加到Transformations列表中。
2. 调用execute，则从sink开始将Transformation转换为StreamNode，同时建立上下游关系StreamEdge。
3. 最终生成StreamGraph。


### Client

Client模块的入口为CliFrontend,用于接收和处理各种命令与请求,如Run和Cancel代表运行和取消任务,CliFrontend在收到对应命令后,根据参数来具体执行命令.
Run命令中必须执行Jar和Class,也可指定SavePoint目录来恢复任务.

Client会根据Jar来提取出Plan,即DataFlow.然后在此Plan的基础上生成JobGraph.其主要操作是对StreamGraph进行优化,将能chain在一起的Operator进行Chain在一起的操作.
在得到JobGraph后就会提交JobGraph等内容,为任务的运行做准备.
Operator能chain在一起的条件:

 1. 上下游Operator的并行度一致

 2. 下游节点的入度为1

 3. 上下游节点都在同一个SlotSharingGroup中(默认为default)

 4. 下游节点的chain策略是ALWAYS(可以与上下游链接,map/flatmap/filter等默认是ALWAYS)

 5. 上游节点的chain策略是ALWAYS或HEAD(只能与下游链接,不能与上游链接,source默认是HEAD)

 6. 两个Operator间的数据分区方式是fowward

 7. 用户没有禁用chain


针对CliFrontend，其会将命令行参数进行解析。如果命令行传参如下:

```conf
run -c org.apache.flink.streaming.examples.WordCount ./WordCount.jar
```

则启动CliFrontend的main方法之后，会检测到为run模式，并且执行的是./WordCount.jar，其入口类为org.apache.flink.streaming.examples.WordCount。然后就开始了向jobGraph的转换。
