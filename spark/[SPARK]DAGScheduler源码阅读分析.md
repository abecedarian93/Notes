### DAGScheduler源码阅读分析

### DAGScheduler的变量:
````
private[spark] val metricsSource: DAGSchedulerSource = new DAGSchedulerSource(this) //统计相关
private[scheduler] val nextJobId = new AtomicInteger(0) // 下一个jobId
private[scheduler] def numTotalJobs: Int = nextJobId.get() // Job总数
private val nextStageId = new AtomicInteger(0) // 下一个StageId
//记录每个job对应的包含的所有stageId
private[scheduler] val jobIdToStageIds = new HashMap[Int, HashSet[Int]]
//记录StageId对应的Stage
private[scheduler] val stageIdToStage = new HashMap[Int, Stage]

/**
  * 从shuffle依赖ID映射到将为该依赖生成数据的ShuffleMapStage
  * 仅包括当前正在运行作业(job)的一部分的stages(当需要随机播放阶段的作业完成时,映射将被删除,并且随机数据的唯一记录将在MapOutputTracker中)
  */
//记录shuffle对应的ShuffleMapStage
private[scheduler] val shuffleIdToMapStage = new HashMap[Int, ShuffleMapStage]
//记录处于Active状态的job,jobId=>ActiveJob
private[scheduler] val jobIdToActiveJob = new HashMap[Int, ActiveJob]
//等待运行的Stage，他们的父Stage还没有运行完成
private[scheduler] val waitingStages = new HashSet[Stage]
//处于Running状态的Stage
private[scheduler] val runningStages = new HashSet[Stage]
//等待重新提交的Stage，原因为fetch failures
private[scheduler] val failedStages = new HashSet[Stage]
//Active状态的Job列表
private[scheduler] val activeJobs = new HashSet[ActiveJob]
//RDD id对应的位置信息，通过Partition号码进行索引，得到的数组是对应的Partition的存储位置信息
private val cacheLocs = new HashMap[Int, IndexedSeq[Seq[TaskLocation]]]
//为了跟踪失败的节点,我们使用MapOutputTracker的epoch编号,该编号随每个任务一起发送
//当我们检测到节点发生故障时,我们会记下当前的epoch编号和失败的executor,为新任务增加它的值,并使用它来忽略杂散的ShuffleMapTask结果
private val failedEpoch = new HashMap[String, Long]

/**
  * sparkEnv环境和sparkConfig配置读取
  */
//sparkEnv初始化的输出提交控制器
private [scheduler] val outputCommitCoordinator = env.outputCommitCoordinator
//我们重用的闭包序列化器,这将只是安全的因为DAGScheduler是运行在一个单线程上的
private val closureSerializer = SparkEnv.get.closureSerializer.newInstance()
//如果启用,FetchFailed将不会导致阶段重试,以解决问题
private val disallowStageRetryForTest = sc.getConf.getBoolean("spark.test.noStageRetry", false)
//收到FetchFailure的情况下是否取消注册主机上的所有输出
private[scheduler] val unRegisterOutputOnHostOnFetchFailure = sc.getConf.get(config.UNREGISTER_OUTPUT_ON_HOST_ON_FETCH_FAILURE)
//在中止stage之前允许的连续stage尝试次数,默认为4
private[scheduler] val maxConsecutiveStageAttempts = sc.getConf.getInt("spark.stage.maxConsecutiveAttempts",DAGScheduler.DEFAULT_MAX_CONSECUTIVE_STAGE_ATTEMPTS)
//每个障碍作业的最大并发任务数检查失败
private[scheduler] val barrierJobIdToNumTasksCheckFailures = new ConcurrentHashMap[Int, Int]
//在最大并发任务检查失败和下一次检查之间等待的时间(以秒为单位)
private val timeIntervalNumTasksCheck = sc.getConf.get(config.BARRIER_MAX_CONCURRENT_TASKS_CHECK_INTERVAL)
//在作业提交失败之前,最大并发任务的最大数量检查作业允许的失败
private val maxFailureNumTasksCheck = sc.getConf.get(config.BARRIER_MAX_CONCURRENT_TASKS_CHECK_MAX_FAILURES)


private val messageScheduler = ThreadUtils.newDaemonSingleThreadScheduledExecutor("dag-scheduler-message")

 // Scheduler事件处理
private[spark] val eventProcessLoop = new DAGSchedulerEventProcessLoop(this)
````

### job提交:
RDD action操作会导致job提交,其内部的调用顺序是:

##### 1. rdd action内部调用SparkContext.runJob函数
````
//rdd调用action操作会导致job提交,调用SparkContext.runJob函数
def runJob[T, U: ClassTag](
                            rdd: RDD[T],
                            func: (TaskContext, Iterator[T]) => U,
                            partitions: Seq[Int],
                            resultHandler: (Int, U) => Unit): Unit = {
  if (stopped.get()) {
    throw new IllegalStateException("SparkContext has been shutdown")
  }
  val callSite = getCallSite
  val cleanedFunc = clean(func)
  logInfo("Starting job: " + callSite.shortForm)
  if (conf.getBoolean("spark.logLineage", false)) {
    logInfo("RDD's recursive dependencies:\n" + rdd.toDebugString)
  }
  dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
  progressBar.foreach(_.finishAll())
  rdd.doCheckpoint()
}
````
其中会调用`dagScheduler.runJob`函数

##### 2. SparkContext.runJob函数调用DagScheduler.runJob函数
````
//在给定的RDD上运行action作业,并在结果到达时将所有结果传递给resultHandler函数
def runJob[T, U](
    rdd: RDD[T],
    func: (TaskContext, Iterator[T]) => U,
    partitions: Seq[Int],
    callSite: CallSite,
    resultHandler: (Int, U) => Unit,
    properties: Properties): Unit = {
  val start = System.nanoTime
  val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
  ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)
  waiter.completionFuture.value.get match {
    case scala.util.Success(_) =>
      logInfo("Job %d finished: %s, took %f s".format
        (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
    case scala.util.Failure(exception) =>
      logInfo("Job %d failed: %s, took %f s".format
        (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
      // SPARK-8644: Include user stack trace in exceptions coming from DAGScheduler.
      val callerStackTrace = Thread.currentThread().getStackTrace.tail
      exception.setStackTrace(exception.getStackTrace ++ callerStackTrace)
      throw exception
  }
}
````
runJob函数内部调用submitJob函数,将action job提交给dagScheduler,submitJob函数主要流程:
- 检查rdd的partition数目正确
- 给当前job生成一个jobId
- 构造一个JobWaiter对象,返回给上一级函数来监控Job的执行状态
- 提交JobSubmitted事件

````
//提交action作业(job)到sheduler
  def submitJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): JobWaiter[U] = {
    // Check to make sure we are not launching a task on a partition that does not exist.
    //检查rdd的partition数目正确
    val maxPartitions = rdd.partitions.length
    partitions.find(p => p >= maxPartitions || p < 0).foreach { p =>
      throw new IllegalArgumentException(
        "Attempting to access a non-existent partition: " + p + ". " +
          "Total number of partitions: " + maxPartitions)
    }

    //给当前job生成一个jobId
    val jobId = nextJobId.getAndIncrement()
    if (partitions.size == 0) {
      // Return immediately if the job is running 0 tasks
      return new JobWaiter[U](this, jobId, 0, resultHandler)
    }

    assert(partitions.size > 0)
    val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
    //构造一个JobWaiter对象,返回给上一级函数来监控Job的执行状态
    val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)
    //提交JobSubmitted事件
    eventProcessLoop.post(JobSubmitted(
      jobId, rdd, func2, partitions.toArray, callSite, waiter,
      SerializationUtils.clone(properties)))
    waiter
  }
````

有两个对象在这里需要说明:`DAGSchedulerEvent`和`JobWaiter`

> `DAGSchedulerEvent`类型的事件都是由`DAGScheduler`内部的`DAGSchedulerEventProcessLoop`处理,它使用消息队列的架构,内部有一个`LinkedBlockingDeque`类型消息队列,post(DAGSchedulerEvent)会将事件添加到队列中,名称为dag-scheduler-event-loop事件线程从阻塞队列中取出事件,调用DAGSchedulerEventProcessLoop.onReceive(event: DAGSchedulerEvent)进行处理,最终调用`DAGSchedule.handleJobSubmitted`方法,`JobSubmitted`是继承了`DAGSchedulerEvent`的一个提交作业的事件

> `JobWaiter`实现了JobListener接口,等待`DAGSchedule`中的job计算完成,当每个task结束后,通过回调函数,将对应结果传递给句柄函数`resultHandler`处理,当所有tasks都完成时就认为job完成


### stage划分:

###### stage:
stage(阶段)本身是一个taskset(task集合),各个task的计算函数都是一样的,只是作用于RDD不同的partition(分区)

stage之间以shuffle作为边界,必须等待前面的stage计算完成才能获取数据

###### stage dependency(依赖关系):
父RDD与子RDD之间存在依赖关系,有两种依赖关系:`NarrowDependency`(窄依赖)和`ShuffleDependency`(宽依赖),DAGScheduler根据宽依赖将job划分不同的Stage
- 子RDD的每个partition只依赖父RDD特定的partition,为窄依赖
- 子RDD的每个partition通常依赖父RDD整个patitions,为宽依赖,需要父RDD操作完成才能开始子RDD的操作

###### stage子类:
stage有两个具体子类`ResultStage`和`ShuffleMapStage`

`ShuffleMapStage`：
- 属于中间阶段,作为其他stage的输入
- 以shuffle操作边界,即依赖为ShuffleDependency(宽依赖)的RDD之前必然是另一个ShuffleMapStage
- stage内部的转换操作(map.filte等)会组成管线,连在一起计算
- 会保存map输出文件

`ResultStage`:
- 计算spark的action操作,产生一个结果返回给driver程序或者写到存储中
- 一个job中只有一个`ResultStage`,且一定是最后一个stage

###### ActiveJob:
在DAGSchedule中job对应的类为ActiveJob,有两种类型的job
- result job:finalStage类型为ResultStage,执行一个action
- map-stage job:finalStage类型为ShufMapStage,计算它的map output

###### handleJobSubmitted(job提交处理):
stage划分是从最后一个RDD开始的,即最后触发action的RDD
- 创建job的最后一个stage,类型ResultStage,同时会向前回朔,创建之前的所有祖先stage
- 创建job,告诉SparkListener该job已经开始执行
- 把创建好的stage提交

````
private[scheduler] def handleJobSubmitted(jobId: Int,
      finalRDD: RDD[_],
      func: (TaskContext, Iterator[_]) => _,
      partitions: Array[Int],
      callSite: CallSite,
      listener: JobListener,
      properties: Properties) {
    var finalStage: ResultStage = null
    try {
      // New stage creation may throw an exception if, for example, jobs are run on a
      // HadoopRDD whose underlying HDFS files have been deleted.
      //如果在已删除其基础HDFS文件的HadoopRDD上运行job,则新stage的创建可能会引发异常
      // 内部会创建job所有的stage
      finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
    } catch {
      case e: BarrierJobSlotsNumberCheckFailed =>
        logWarning(s"The job $jobId requires to run a barrier stage that requires more slots " +
          "than the total number of slots in the cluster currently.")
        // If jobId doesn't exist in the map, Scala coverts its value null to 0: Int automatically.
        val numCheckFailures = barrierJobIdToNumTasksCheckFailures.compute(jobId,
          new BiFunction[Int, Int, Int] {
            override def apply(key: Int, value: Int): Int = value + 1
          })
        if (numCheckFailures <= maxFailureNumTasksCheck) {
          messageScheduler.schedule(
            new Runnable {
              override def run(): Unit = eventProcessLoop.post(JobSubmitted(jobId, finalRDD, func,
                partitions, callSite, listener, properties))
            },
            timeIntervalNumTasksCheck,
            TimeUnit.SECONDS
          )
          return
        } else {
          // Job failed, clear internal data.
          barrierJobIdToNumTasksCheckFailures.remove(jobId)
          listener.jobFailed(e)
          return
        }

      case e: Exception =>
        logWarning("Creating new stage failed due to exception - job: " + jobId, e)
        listener.jobFailed(e)
        return
    }
    // Job submitted, clear internal data.
    //job提交,清除内部数据
    barrierJobIdToNumTasksCheckFailures.remove(jobId)

    // 创建job
    val job = new ActiveJob(jobId, finalStage, callSite, listener, properties)//创建一个计算ResultStage类型的ActiveJob
    clearCacheLocs()//清除对RDD每个partition缓存位置的记录信息
    logInfo("Got job %s (%s) with %d output partitions".format(
      job.jobId, callSite.shortForm, partitions.length))
    logInfo("Final stage: " + finalStage + " (" + finalStage.name + ")")
    logInfo("Parents of final stage: " + finalStage.parents)
    //查看finalStage的父stage是否已经有计算结果
    logInfo("Missing parents: " + getMissingParentStages(finalStage))

    val jobSubmissionTime = clock.getTimeMillis()
    jobIdToActiveJob(jobId) = job//jobId和ActiveJob的映射
    activeJobs += job
    finalStage.setActiveJob(job)
    val stageIds = jobIdToStageIds(jobId).toArray
    val stageInfos = stageIds.flatMap(id => stageIdToStage.get(id).map(_.latestInfo))
    listenerBus.post(//获取job和stage信息,通过消息总线,告诉SparkListener该Job已经开始了
      SparkListenerJobStart(job.jobId, jobSubmissionTime, stageInfos, properties))
    submitStage(finalStage)//提交stage,进行下一步处理
  }
````

###### createResultStage函数:
创建job最后一个stage:`ResultStage`,其构造函数要求获取它依赖的父stage,这是进行stage切分的核心,内部从该stage开始向前回溯,遇到`shuffleDependency`就进行切分,如果是`narrowDependency`就归并到同一个stage
````
//创建与jobId相关联的ResultStage(ResultStage内部包含之前的所有祖先stage)
private def createResultStage(
    rdd: RDD[_],
    func: (TaskContext, Iterator[_]) => _,
    partitions: Array[Int],
    jobId: Int,
    callSite: CallSite): ResultStage = {
  checkBarrierStageWithDynamicAllocation(rdd)
  checkBarrierStageWithNumSlots(rdd)
  checkBarrierStageWithRDDChainPattern(rdd, partitions.toSet.size)
  val parents = getOrCreateParentStages(rdd, jobId)//获取或者创建该job最后一个stage:ResultStage之前的stage
  val id = nextStageId.getAndIncrement()//生成一个stageId
  val stage = new ResultStage(id, rdd, func, partitions, parents, jobId, callSite)
  stageIdToStage(id) = stage//记录stageId和stage的对应关系
  updateJobIdStageIdMaps(jobId, stage)//记录JobId和stage的对应关系
  stage
}
````

###### getOrCreateParentStages函数:
找到该RDD的ShuffleDependency,获取对应的ShuffleMapStage

从当前RDD开始使用DFS回朔并进行标记,直到对应路径上遍历到ShuffleDependency或RDD都遍历完成为止
````
//获取或创建给定RDD的所有祖先stage,使用提供的firstJobId创建新的stage
private def getOrCreateParentStages(rdd: RDD[_], firstJobId: Int): List[Stage] = {
  getShuffleDependencies(rdd).map { shuffleDep =>
    getOrCreateShuffleMapStage(shuffleDep, firstJobId)
  }.toList
}

/**
    *返回指定RDD的直接父项的shuffle-dependencies
    *
    *此功能不会返回更远的祖先.例如,如果C对B有一个shuffle-dependency,它对A有一个shuffle-dependency:
    * 
    *A <-- B <-- C
    *
    *使用rdd C调用此函数只会返回 B <-- C dependency
    */
private[scheduler] def getShuffleDependencies(
    rdd: RDD[_]): HashSet[ShuffleDependency[_, _, _]] = {
  val parents = new HashSet[ShuffleDependency[_, _, _]]
  val visited = new HashSet[RDD[_]]
  val waitingForVisit = new ArrayStack[RDD[_]]
  waitingForVisit.push(rdd)
  while (waitingForVisit.nonEmpty) {
    val toVisit = waitingForVisit.pop()
    if (!visited(toVisit)) {
      visited += toVisit
      toVisit.dependencies.foreach {
        case shuffleDep: ShuffleDependency[_, _, _] =>
          parents += shuffleDep
        case dependency =>
          waitingForVisit.push(dependency.rdd)
      }
    }
  }
  parents
}
````

###### getOrCreateShuffleMapStage函数:
````
//如果shuffleIdToMapStage中存在一个shuffleMapStage,则获取它
//如果shuffleMapStage尚不存在,则除了任何缺少的祖先shuffleMapStage之外,此方法还将创建shuffleMapStage
private def getOrCreateShuffleMapStage(
     shuffleDep: ShuffleDependency[_, _, _],
     firstJobId: Int): ShuffleMapStage = {
   shuffleIdToMapStage.get(shuffleDep.shuffleId) match {
     case Some(stage) =>
       stage

     case None =>
       // Create stages for all missing ancestor shuffle dependencies.
       getMissingAncestorShuffleDependencies(shuffleDep.rdd).foreach { dep =>
         // Even though getMissingAncestorShuffleDependencies only returns shuffle dependencies
         // that were not already in shuffleIdToMapStage, it's possible that by the time we
         // get to a particular dependency in the foreach loop, it's been added to
         // shuffleIdToMapStage by the stage creation process for an earlier dependency. See
         // SPARK-13902 for more information.
         // 再次检查避免重复创建(多个ShuffleDependency又依赖到同一个ShuffleDependency的情况)
         if (!shuffleIdToMapStage.contains(dep.shuffleId)) {
           createShuffleMapStage(dep, firstJobId)
         }
       }
       // Finally, create a stage for the given shuffle dependency.
       //最后,为指定的shuffle-dependency创建一个stage
       createShuffleMapStage(shuffleDep, firstJobId)
   }
 }
 
//找到还未在shuffleToMapStage中注册的ancestors(祖先)shuffleDependencies
private def getMissingAncestorShuffleDependencies(
    rdd: RDD[_]): ArrayStack[ShuffleDependency[_, _, _]] = {
  val ancestors = new ArrayStack[ShuffleDependency[_, _, _]]
  val visited = new HashSet[RDD[_]]
  // We are manually maintaining a stack here to prevent StackOverflowError
  // caused by recursively visiting
  // stack结构用来从最开始创建stage
  val waitingForVisit = new ArrayStack[RDD[_]]
  waitingForVisit.push(rdd)
  while (waitingForVisit.nonEmpty) {
    val toVisit = waitingForVisit.pop()
    if (!visited(toVisit)) {
      visited += toVisit
      getShuffleDependencies(toVisit).foreach { shuffleDep =>
        if (!shuffleIdToMapStage.contains(shuffleDep.shuffleId)) {
          ancestors.push(shuffleDep)
          waitingForVisit.push(shuffleDep.rdd)
        } // Otherwise, the dependency and its ancestors have already been registered.
        //否则,dependency及其ancestors(祖先)已经注册
      }
    }
  }
  ancestors
}

//创建一个ShuffleMapStage,生成给定的shuffleDependency的分区
//如果先前运行的阶段生成相同的shuffle数据,则此函数将复制仍可从先前的shuffle获得的输出位置,以避免不必要地重新生成数据
def createShuffleMapStage(shuffleDep: ShuffleDependency[_, _, _], jobId: Int): ShuffleMapStage = {
  val rdd = shuffleDep.rdd //该Stage最后一个RDD,用来进行map端的shuffle
  checkBarrierStageWithDynamicAllocation(rdd)
  checkBarrierStageWithNumSlots(rdd)
  checkBarrierStageWithRDDChainPattern(rdd, rdd.getNumPartitions)
  // ShuffleMapStage中task的数目与rdd分区数目相同
  val numTasks = rdd.partitions.length
  // 该Stage依赖的父Stage,所以这里需要确保Stage从前往后创建
  val parents = getOrCreateParentStages(rdd, jobId)
  // 获取一个stageID，从0开始
  val id = nextStageId.getAndIncrement()
  val stage = new ShuffleMapStage(
    id, rdd, numTasks, parents, jobId, rdd.creationSite, shuffleDep, mapOutputTracker)

  stageIdToStage(id) = stage//记录stageId和stage的对应关系
  shuffleIdToMapStage(shuffleDep.shuffleId) = stage//记录shuffleId和stage的对应关系
  updateJobIdStageIdMaps(jobId, stage)//记录JobId和stageID的对应关系(Stage的jobIds字段，DAGSchedule的jobIdToStageIds字段)

  //注册mapOutputTracker，用来记录createShuffleMapStaged的输出
  if (!mapOutputTracker.containsShuffle(shuffleDep.shuffleId)) {
    // Kind of ugly: need to register RDDs with the cache and map output tracker here
    // since we can't do it in the RDD constructor because # of partitions is unknown
    logInfo("Registering RDD " + rdd.id + " (" + rdd.getCreationSite + ")")
    mapOutputTracker.registerShuffle(shuffleDep.shuffleId, rdd.partitions.length)
  }
  stage
}
````

### 提交stage
提交stage,先递归提交未计算出的parent stage计算

确认stage是否需要计算的关键是看该stage对应的最后一个RDD,如果该RDD的所有partitions都可以从blockManager中获取到位置,就不用再次计算该计算,否则重新计算该stage

````
//提交stage,但首先会递归提交任何缺少的parents stage
private def submitStage(stage: Stage) {
  val jobId = activeJobForStage(stage)
  if (jobId.isDefined) {
    logDebug("submitStage(" + stage + ")")
    // 当前stage没有处于等待，运行或者失败状态
    if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
      //回朔Rdd,取决于其对应的StorageLevel，是否可以从BlockManager中获取到对应RddBlock，如果碰到不能获取，且其依赖为ShuffleDependency，就判定为缺失
      val missing = getMissingParentStages(stage).sortBy(_.id)
      logDebug("missing: " + missing)
      if (missing.isEmpty) {
        logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
        submitMissingTasks(stage, jobId.get)
      } else {
        //有parent stage未完成,递归提交,最后把自身添加到等待队列中
        for (parent <- missing) {
          submitStage(parent)
        }
        waitingStages += stage
      }
    }
  } else {
    abortStage(stage, "No active job for stage " + stage.id, None)
  }
}
````

### 提交task(任务)

###### submitMissingTasks
把stage需要计算的分区转换为TaskSet,通过TaskScheduler.submitTasks(taskSet: TaskSet)提交
- 确定需要计算的分区 `partitionsToCompute: Seq[Int]`
- 确定每个分区的位置信息 `taskIdToLocations: Map[Int, Seq[TaskLocation]]`
- 创建`StageInfo`
- 创建广播`taskBinary`,`ShuffleMapStage`广播(rdd,shuffleDep),`ResultTask`广播(rdd, func)
- 创建`tasks:Seq[Task[_]]`,`ShuffleMapTask`或者`ResultTask`
- 发送`taskScheduler.submitTasks(TaskSet)`

````
//当stage的parents可用时,我们现在就可以执行它的task
private def submitMissingTasks(stage: Stage, jobId: Int) {
  logDebug("submitMissingTasks(" + stage + ")")

  // First figure out the indexes of partition ids to compute.
  //首先找出要计算的partition Ids的indexes(索引)
  //ShuffleMapStage 通过MapOutputTrackerMaster获得
  //ResultStage 通过ActiveJob获取
  val partitionsToCompute: Seq[Int] = stage.findMissingPartitions()

  // Use the scheduling pool, job group, description, etc. from an ActiveJob associated
  // with this Stage
  //job对应的属性,主要是调度池的名称
  val properties = jobIdToActiveJob(jobId).properties

  //标识为正在运行中的Stage
  runningStages += stage
  // SparkListenerStageSubmitted should be posted before testing whether tasks are
  // serializable. If tasks are not serializable, a SparkListenerStageCompleted event
  // will be posted, which should always come after a corresponding SparkListenerStageSubmitted
  // event.
  //OutputCommitCoordinator记录stage每个partition的信息,用来决定task是否有权利进行输出
  stage match {
    case s: ShuffleMapStage =>
      outputCommitCoordinator.stageStart(stage = s.id, maxPartitionId = s.numPartitions - 1)
    case s: ResultStage =>
      outputCommitCoordinator.stageStart(
        stage = s.id, maxPartitionId = s.rdd.partitions.length - 1)
  }
  //获取该stage每个taskId的位置信息
  //首先是查找缓存信息,然后是checkpiont信息,RDD自身的preferredLocations,如果是NarrowDependency,读取父RDD对应partition的位置信息
  val taskIdToLocations: Map[Int, Seq[TaskLocation]] = try {
    stage match {
      case s: ShuffleMapStage =>
        partitionsToCompute.map { id => (id, getPreferredLocs(stage.rdd, id))}.toMap
      case s: ResultStage =>
        partitionsToCompute.map { id =>
          val p = s.partitions(id)
          (id, getPreferredLocs(stage.rdd, p))
        }.toMap
    }
  } catch {
    case NonFatal(e) =>
      stage.makeNewStageAttempt(partitionsToCompute.size)
      listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo, properties))
      abortStage(stage, s"Task creation failed: $e\n${Utils.exceptionString(e)}", Some(e))
      runningStages -= stage
      return
  }

  //创建一个包含各种LongAccumulator字段,用于统计各种性能耗时的TaskMetrics,然后创建StageInfo,记录所有该stage相关的信息
  stage.makeNewStageAttempt(partitionsToCompute.size, taskIdToLocations.values.toSeq)

  // If there are tasks to execute, record the submission time of the stage. Otherwise,
  // post the even without the submission time, which indicates that this stage was
  // skipped.
  //设置StageInfo中stage从DAGScheduler提交到TaskScheduler的时间
  if (partitionsToCompute.nonEmpty) {
    stage.latestInfo.submissionTime = Some(clock.getTimeMillis())
  }
  //向消息总线传递SparkListenerStageSubmitted事件
  listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo, properties))

  // TODO: Maybe we can keep the taskBinary in Stage to avoid serializing it multiple times.
  // Broadcasted binary for the task, used to dispatch tasks to executors. Note that we broadcast
  // the serialized copy of the RDD and for each task we will deserialize it, which means each
  // task gets a different copy of the RDD. This provides stronger isolation between tasks that
  // might modify state of objects referenced in their closures. This is necessary in Hadoop
  // where the JobConf/Configuration object is not thread-safe.
  //task相关信息串行化,包括RDD和对应作用到RDD上的函数
  var taskBinary: Broadcast[Array[Byte]] = null
  var partitions: Array[Partition] = null
  try {
    // For ShuffleMapTask, serialize and broadcast (rdd, shuffleDep).
    // For ResultTask, serialize and broadcast (rdd, func).
    var taskBinaryBytes: Array[Byte] = null
    // taskBinaryBytes and partitions are both effected by the checkpoint status. We need
    // this synchronization in case another concurrent job is checkpointing this RDD, so we get a
    // consistent view of both variables.
    RDDCheckpointData.synchronized {
      taskBinaryBytes = stage match {
        case stage: ShuffleMapStage =>
          JavaUtils.bufferToArray(
            closureSerializer.serialize((stage.rdd, stage.shuffleDep): AnyRef))
        case stage: ResultStage =>
          JavaUtils.bufferToArray(closureSerializer.serialize((stage.rdd, stage.func): AnyRef))
      }

      partitions = stage.rdd.partitions
    }

    taskBinary = sc.broadcast(taskBinaryBytes)
  } catch {
    // In the case of a failure during serialization, abort the stage.
    case e: NotSerializableException =>
      abortStage(stage, "Task not serializable: " + e.toString, Some(e))
      runningStages -= stage

      // Abort execution
      return
    case e: Throwable =>
      abortStage(stage, s"Task serialization failed: $e\n${Utils.exceptionString(e)}", Some(e))
      runningStages -= stage

      // Abort execution
      return
  }

  //创建task
  val tasks: Seq[Task[_]] = try {
    val serializedTaskMetrics = closureSerializer.serialize(stage.latestInfo.taskMetrics).array()
    stage match {
      //ShuffleMapStage 生成 ShuffleMapTask
      case stage: ShuffleMapStage =>
        stage.pendingPartitions.clear()
        partitionsToCompute.map { id =>
          val locs = taskIdToLocations(id)
          val part = partitions(id)
          stage.pendingPartitions += id
          new ShuffleMapTask(stage.id, stage.latestInfo.attemptNumber,
            taskBinary, part, locs, properties, serializedTaskMetrics, Option(jobId),
            Option(sc.applicationId), sc.applicationAttemptId, stage.rdd.isBarrier())
        }
      //ResultStage 生成 ResultTask
      case stage: ResultStage =>
        partitionsToCompute.map { id =>
          val p: Int = stage.partitions(id)
          val part = partitions(p)
          val locs = taskIdToLocations(id)
          new ResultTask(stage.id, stage.latestInfo.attemptNumber,
            taskBinary, part, locs, id, properties, serializedTaskMetrics,
            Option(jobId), Option(sc.applicationId), sc.applicationAttemptId,
            stage.rdd.isBarrier())
        }
    }
  } catch {
    case NonFatal(e) =>
      abortStage(stage, s"Task creation failed: $e\n${Utils.exceptionString(e)}", Some(e))
      runningStages -= stage
      return
  }

  if (tasks.size > 0) {
    logInfo(s"Submitting ${tasks.size} missing tasks from $stage (${stage.rdd}) (first 15 " +
      s"tasks are for partitions ${tasks.take(15).map(_.partitionId)})")
    //把task序列转换为TaskSet,提交给taskScheduler
    taskScheduler.submitTasks(new TaskSet(
      tasks.toArray, stage.id, stage.latestInfo.attemptNumber, jobId, properties))
  } else {
    // Because we posted SparkListenerStageSubmitted earlier, we should mark
    // the stage as completed here in case there are no tasks to run
    //如果task数目为0,该Stage可以从running stages中移除,并且把之前等待该stage的子stage取出并提交,提交
    markStageAsFinished(stage, None)

    stage match {
      case stage: ShuffleMapStage =>
        logDebug(s"Stage ${stage} is actually done; " +
            s"(available: ${stage.isAvailable}," +
            s"available outputs: ${stage.numAvailableOutputs}," +
            s"partitions: ${stage.numPartitions})")
        markMapStageJobsAsFinished(stage)
      case stage : ResultStage =>
        logDebug(s"Stage ${stage} is actually done; (partitions: ${stage.numPartitions})")
    }
    submitWaitingChildStages(stage)
  }
}
````




