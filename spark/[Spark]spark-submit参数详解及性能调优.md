### Spark-submit参数详解及参数调优

#### 1.spark-submit参数详解
###### 可以通过spark-submit --help来查看这些参数

> 使用demo:
````
# Run on a YARN cluster
export HADOOP_CONF_DIR=XXX
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master yarn \
  --deploy-mode cluster \  # can be client for client mode
  --executor-memory 20G \
  --num-executors 50 \
  /path/to/examples.jar \
  1000
````
> 参数说明
 
参数 | 说明 | 参考值 |默认值
--- |---|---|---
--class | 作业的主类 | org.apache.spark.examples.SparkPi|-
--master| 作业运行模式 |spark://host:port,mesos://host:port,yarn,yarn-cluster,yarn-client,local|-
--deploy-mode| client表示driver程序运行在提交作业的节点,cluster表示driver程序运行在cluster其中一台机器上面|client,cluster|-
--name| 作业的名字 |-|-
--jars| 添加jars到driver和executor的classpaths中,多个jar逗号分隔|-|-
--packages | 添加maven坐标(groupId:artifactId:version)到driver和executor的classpaths中,多个maven坐标逗号分隔|-|-
--exclude-packages |添加用逗号分隔的(groupId:artifactId)列表排除--packages的依赖以解决冲突|-|-
--repositories | 逗号分隔的列表添加maven仓库用于寻找--packages给出的maven坐标|-|-
--py-files| 逗号分隔的列表(.zip,.egg,.py文件)用于替代PYTHONPATH|-|-
--files |上传文件到executor工作区域,多个文件用逗号分隔|-|-
--conf |spark配置属性|--conf spark.driver.maxResultSize=4g|-|-
--properties-file|加载额外的配置文件|-|conf/spark-defaults.conf
--driver-memory|driver内存大小|(e.g. 1000M, 2G)|1024M
--driver-java-options|driver端java额外选项|-|-
--driver-library-path|driver端额外库路径|-|-
--driver-class-path|driver端额外类路径(--jars参数所属路径自动加载到这个路径)|-|-
--executor-memory|executor内存大小|(e.g. 1000M, 2G)|1G
--proxy-user|模拟提交应用程序的用户(不能和--principal / --keytab一起使用)|-|-
--driver-cores|driver的核数,参数仅在standalone或yarn cluster模式下使用|e.g. 2,4|1
--supervise|driver失败后,重启,参数在standalone或Mesos cluster模式下使用|-|-
--kill |杀掉driver,参数在standalone或Mesos cluster模式下使用|-|-
--status |查询driver状态|-|-
--total-executor-cores|所有executor总共的核数,仅在standalone或Mesos使用|-|-
--executor-cores|每个executor核数|e.g. 100,200|yarn模式为1,standalone模式为所有可用核数
--queue |在yarn模型下队列名称|-|default
--num-executors |启动executor的数量,在yarn使用,如果允许动态分配,该值为executor的至少数量|-|2
--archives |在executor工作区域可提取的文档,多个文档以逗号分隔|-|-
--principal |在yarn模式下用于登陆KDC|-|-
--keytab|在yarn模式下包含keytab文件的全路径|-|-


#### 2.spark-on-yarn 参数调优
> 通过调节各种参数,优化spark-on-yarn使用各个资源的效率,提升执行效率
> 参数调优

参数|参数说明|调优建议
---|---|---
num-executors|申请executor数量,如果不设置,driver向yarn集群申请的executor默认为2个,spark作业运行会超级慢|每个作业申请50～200个左右都较为合理,结合业务的实际情况,太少资源利用不充分,太多可能无法给予满足,占用资源太多也会导致其他作业等待
executor-memory|每个executor的内存大小,如果JVM OOM异常,直接原因是内存不足,内存大小基本直接决定了作业的性能|每个executor内存申请4G~10G都可以,默认1G在多数情况是不够的.需要注意的是yarn会对每个队列最大内存作限制,num-executors*executor-memory不能超过该限制,队列是公用的,建议不要超过最大内存限制1/2
executor-cores|每个executor的cpu core数量,决定executor并行执行task线程的能力,该值越大,越能快速执行完分配的task任务|每个executor核数建议2～4个,yarn会对每个队列最大的cpu core数量做限制,建议不要超过最大核数限制1/2
driver-memory|driver端的内存大小|默认1G在大多数情况不需要调整的,如果需要将数据拉取到driver端时,需要增加该值,避免出现OOM内存溢出.collect算子,广播变量较大时(map-join很多时候把小表当作广播变量)等操作会将RDD拉取到driver端
spark.default.parallelism|每个stage的默认task数量,很主要很初学者懵逼的参数,很多情况下默认值会降低作业性能|该值的默认值是分块的个数(HDFS的block数量),默认值较少导致设置的executor cpu资源闲置(task只有10,cpu100个core,90%的core浪费掉了),建议该值为num-executors * executor-cores(cpu总core数)的2~3倍
spark.storage.memoryFraction|rdd持久化数据在executor内存的占比,默认为0.6|作业中如果较少的持久化操作,60%的executor内存显得不那么合理,可酌情降低该比例,反之较多持久化时,需要增大该比例
spark.shuffle.memoryFraction|shuffle过程task拉取上个stage的输出后,进行聚合操作用到的executor内存占比,默认为0.2,使用的内存超出会发生溢写,产生的IO降低性能|作业中如果较多的shuffle,较少的持久化操作,可增大该值,降低spark.storage.memoryFraction(持久化内存占比)值,当然如果作业频繁gc运行缓慢,除增加executor的内存,也需要考虑降低持久化内存占比和该值(shuffle内存占比)


###### 相关知识点:
> maven坐标：groupId:artifactId:version 确定资源唯一

###### 相关资料链接:

- https://www.cnblogs.com/camilla/p/8301750.html
- https://spark.apache.org/docs/latest/running-on-yarn.html
