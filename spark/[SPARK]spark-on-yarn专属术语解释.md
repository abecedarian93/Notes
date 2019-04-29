### spark-on-yarn专属术语解释

> yarn相关术语
- Yarn: 集群的资源管理系统,hadoop包含yarn,它将资源管理和处理组件分开
- ResourceManager: 负责yarn集群的资源管理和分配
- ApplicationMaster: yarn中每个Application对应一个applicationMaster(AM)进程,负责与resourceManager(RM)协商获取资源,获取资源后告诉NodeManager为其分配并启动Container
- NodeManager: 每个节点的资源和任务管理器,负责启动/停止Container,并监视资源使用情况
- Container: yarn中的抽象资源

> spark相关术语
- Driver: 与clusterManager(CM)通信,进行资源申请、任务分配并监督其运行状况等
- ClusterManager: 集群资源管理,这里指的就是yarn
- DAGScheduler: 把spark作业转换成Stage的DAG图
- TaskScheduler: 把Task分配给具体的Executor
