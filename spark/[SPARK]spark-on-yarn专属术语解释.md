### spark-on-yarn专属术语解释


> yarn相关概念

术语|解释
---|---
Yarn|集群的资源管理系统,hadoop包含yarn,它将资源管理和处理组件分开
ResourceManager|负责yarn集群的资源管理和分配
ApplicationMaster|yarn中每个Application对应一个applicationMaster(AM)进程,负责与resourceManager(RM)协商获取资源,获取资源后告诉nodeManager为其分配并启动container
NodeManager|每个节点的资源和任务管理器,负责启动/停止Container,并监视资源使用情况
Container|yarn中的抽象资源

> spark基本概念

术语|解释
---|---
Driver| 与clusterManager(CM)通信,进行资源申请、任务分配并监督其运行状况,运行用户自己编写的应用程序main函数,创建sparkContext
Client Node|将应用程序提交,负责业务逻辑和master节点通信
Master Node|常驻master进程,负责管理全部worker节点
Worker Node|常驻worker进程,负责管理executor并与master节点通信
ClusterManager|集群资源管理,这里指的就是yarn (其他还有standlone，mesos)
DAGScheduler|把spark作业转换成stage的DAG图
TaskScheduler|把task分配给具体的executor
Executor|(执行器)监控和执行task的容器
Application|用户编写的spark应用程序,由一个或者多个job组成
Job|(作业)每个rdd的action算子产生一个job,包含多个task并行计算,用户提交的job会提交给DAGScheduler,job会被分解成stage和task
Stage|(调度阶段)一个job会被拆分为多个taskSet,每个taskSet在调度阶段称为stage,stage的划分依据是**shuffle之前的所有变换是一个stage，shuffle之后的操作是另一个stage**(较为准确的是根据RDD的宽依赖关系)
TaskSet|(任务集)一组关联的,但相互没有Shuffle依赖关系的Task集合
Task|executor上的任务执行单元,rdd的partition决定task数,task处理一个partition数据.一般task分为shuffleMapTask(输出是shuffle所需数据)和resultTask(输出是result)

> spark基本概念之间关系
- 一个Application可以由一个或者多个job组成,一个job可以由一个或者多个stage组成,其中stage是根据宽窄依赖进行划分的,一个stage由一个taskSet组成,一个taskSet可以由一个到多个task组成

> spark类相关概念

类名|解释
---|---
org.apache.spark.SparkException|spark异常处理类
org.apache.spark.SparkContext|spark功能的主入口,是一个容器,里面装各种各样的资源,用来代表与spark-cluster的连接,可在cluster上创建RDDs, 累加器(accumulators)和广播变量(broadcast variables),目前一个jvm只能存在一个sc(SparkContext),只有干掉(stop()函数)当前sc,才能创建新的.未来的版本可能对该限制修改,详情见[SPARK-2243](https://issues.apache.org/jira/browse/SPARK-2243)
org.apache.spark.SparkConf|spark application的配置类,用键值对(key-value)设置参数
org.apache.spark.SparkEnv|spark的执行环境对象,其中包括序列化(serializer),组件通信的执行环境(RpcEnv),blockManager管理协调(block manager),跟踪map任务输出状态(map output tracker)等,存在于driver或者coarseGrainedExecutorBackend进程中
org.apache.spark.SparkFiles|解析 `SparkContext.addFile()`添加文件路径
org.apache.spark.SparkStatusTracker|监控job和stage的低层次(low-level)状态的api

###### 相关资料链接:
- https://www.jianshu.com/p/014cb82f462a
- https://www.cnblogs.com/chushiyaoyue/p/7093695.html
- https://spark.apache.org/docs/latest/running-on-yarn.html
- https://github.com/apache/spark
- https://www.cnblogs.com/xia520pi/p/8609625.html
