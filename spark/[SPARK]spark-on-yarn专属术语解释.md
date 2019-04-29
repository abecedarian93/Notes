### spark-on-yarn专属术语解释


> yarn相关术语

术语|解释
---|---
Yarn|集群的资源管理系统,hadoop包含yarn,它将资源管理和处理组件分开
ResourceManager|负责yarn集群的资源管理和分配
ApplicationMaster|yarn中每个Application对应一个applicationMaster(AM)进程,负责与resourceManager(RM)协商获取资源,获取资源后告诉NodeManager为其分配并启动Container
NodeManager|每个节点的资源和任务管理器,负责启动/停止Container,并监视资源使用情况
Container|yarn中的抽象资源

> spark相关术语

术语|解释
---|---
Driver| 与clusterManager(CM)通信,进行资源申请、任务分配并监督其运行状况,运行用户自己编写的应用程序
Master Node|常驻master进程,负责管理全部worker节点
Worker Node|常驻worker进程,负责管理executor并与master节点通信
ClusterManager|集群资源管理,这里指的就是yarn (其他还有standlone，mesos)
DAGScheduler|把spark作业转换成stage的DAG图
TaskScheduler|把task分配给具体的executor
Executor|执行器,监控和执行task的容器
Job|每个rdd的action计算产生一个job,包含多个task并行计算,用户提交的job会提交给DAGScheduler,job会被分解成stage和task
Stage|一个job会被拆分为多组task,每组task称为stage,stage的划分依据是**shuffle之前的所有变换是一个stage，shuffle之后的操作是另一个stage**
Task|executor上的任务执行单元,rdd的partition决定task数,task处理一个partition数据.一般task分为shuffleMapTask(输出是shuffle所需数据)和resultTask(输出是result)



