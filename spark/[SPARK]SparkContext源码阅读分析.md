# SparkContext源码阅读分析

### SparkContext概念

Spark功能的主要入口点。 SparkContext表示与一个Spark集群的连接,可用于在该群集上创建RDD,累加器和广播变量

每个JVM只能激活一个SparkContext,必须先通过stop()方法停止活动的SparkContext,然后创建一个新的.这个限制未来会取消,详情请参阅SPARK-2243

参数:Spark Config对象用于描述应用程序的配置,任何对此配置的设置都会覆盖默认值以及系统属性

Spakr集群通常指:
- local
- standalone
- yarn
- mesos


### SparkContext变量:
````
//构建此SparkContext的调用站点(call site)
//方便开发人员查看调用信息,例如构建此SparkContext的类名和代码行数(shortForm和longForm)
private val creationSite: CallSite = Utils.getCallSite()

//为true的话,在多个SparkContexts处于活动状态时记录警告而不是抛出异常,默认暂时不允许多个处于活跃状态的SparkContexts
private val allowMultipleContexts: Boolean =
    config.getBoolean("spark.driver.allowMultipleContexts", false)

//记录开始时间
val startTime = System.currentTimeMillis()

//判断上下文状态,是否停止
private[spark] val stopped: AtomicBoolean = new AtomicBoolean(false)

//以下的这些私有变量, 变量保持上下文的内部状态,并且是外界无法进入
//它们是可变的,因为我们想要提前初始化所有它们的值为一些中性的值(neutral value),以便在调用"stop()"时,构造函数仍在运行是安全的
private var _conf: SparkConf = _ //spark配置类
private var _eventLogDir: Option[URI] = None //事件日志目录
private var _eventLogCodec: Option[String] = None //事件日志压缩格式
private var _listenerBus: LiveListenerBus = _ //事件总线,可以接收各种使用方的事件,并且异步传递Spark事件监听与SparkListeners监听器的注册
private var _env: SparkEnv = _ //Spark的执行环境,所有的线程都可以通过SparkContext访问到同一个SparkEnv对象
private var _statusTracker: SparkStatusTracker = _ //低级别的状态报告API,只能提供非常脆弱的一致性机制,对job(作业)、stage(阶段)的状态进行监控
private var _progressBar: Option[ConsoleProgressBar] = None //显示控制台下一行中stages的进度
private var _ui: Option[SparkUI] = None //为Spark监控Web平台提供了Spark环境、任务的整个生命周期的监控
private var _hadoopConfiguration: Configuration = _ //Spark默认使用HDFS来作为分布式文件系统,用于获取Hadoop配置信息
private var _executorMemory: Int = _ //executor内存大小
private var _schedulerBackend: SchedulerBackend = _ //与master、worker通信收集worker上分配给该应用使用的资源情况
private var _taskScheduler: TaskScheduler = _ //为Spark的任务调度资源分配器,spark通过他提交任务并请求集群调度任务.因其调度的task由DAGScheduler创建,所以DAGScheduler是TaskScheduler的前置调度
private var _heartbeatReceiver: RpcEndpointRef = _ // 心跳接收器,在driver中创建心跳接收器为了对executor进行掌握
@volatile private var _dagScheduler: DAGScheduler = _ //为高级的、基于Stage的调度器,负责创建job,将DAG中的rdd划分到不同的stage,并将stage作为taskSets提交给底层调度器taskScheduler执行
private var _applicationId: String = _ //spark application id
private var _applicationAttemptId: Option[String] = None // application attempt id
private var _eventLogger: Option[EventLoggingListener] = None //将事件持久化到存储的监听器
private var _executorAllocationManager: Option[ExecutorAllocationManager] = None //用于对以分配的Executor进行管理,可以修改属性spark.dynamicAllocation.enabled为true来创建
private var _cleaner: Option[ContextCleaner] = None //用于清理超出应用范围的rdd、shuffleDependency和broadcast对象
private var _listenerBusStarted: Boolean = false //事件总线状态
private var _jars: Seq[String] = _ //jar包
private var _files: Seq[String] = _ //文件
private var _shutdownHookRef: AnyRef = _ //
private var _statusStore: AppStatusStore = _ //提供存储在其中API的方法

````

### SparkContext初始化:
```
try {
    //读取SparkConf的信息,即Spark的配置信息,并检查是否有非法的配置信息
    _conf = config.clone()
    _conf.validateSettings()

    //判断是否配置了Master,没有的话抛出异常
    if (!_conf.contains("spark.master")) {
      throw new SparkException("A master URL must be set in your configuration")
    }
    //判断是否配置了AppName,没有的话抛出异常
    if (!_conf.contains("spark.app.name")) {
      throw new SparkException("An application name must be set in your configuration")
    }

    // log out spark.app.name in the Spark driver logs
    //在Spark driver日志中打印spark.app.name
    logInfo(s"Submitted application: $appName")

    // System property spark.yarn.app.id must be set if user code ran by AM on a YARN cluster
    //如果用户代码在YARN群集上由AM运行,则必须设置系统属性spark.yarn.app.id
    if (master == "yarn" && deployMode == "cluster" && !_conf.contains("spark.yarn.app.id")) {
      throw new SparkException("Detected yarn cluster mode, but isn't running on a cluster. " +
        "Deployment to YARN is not supported directly by SparkContext. Please use spark-submit.")
    }

    if (_conf.getBoolean("spark.logConf", false)) {
      logInfo("Spark configuration:\n" + _conf.toDebugString)
    }

    // Set Spark driver host and port system properties. This explicitly sets the configuration
    // instead of relying on the default value of the config constant.
    //设置Spark driver host和port系统属性.这显式设置配置,而不是依赖配置常量的默认值
    _conf.set(DRIVER_HOST_ADDRESS, _conf.get(DRIVER_HOST_ADDRESS))
    _conf.setIfMissing("spark.driver.port", "0")

    _conf.set("spark.executor.id", SparkContext.DRIVER_IDENTIFIER)

    _jars = Utils.getUserJars(_conf) //获取jars,在yarn运行模式下返回空,yarn有自己获取jars的机制
    _files = _conf.getOption("spark.files").map(_.split(",")).map(_.filter(_.nonEmpty))
      .toSeq.flatten //获取files

    _eventLogDir =
      if (isEventLogEnabled) {
        //这里需要进行设置,否则默认路径是“/tmp/spark-events”,防止系统自动清除
        val unresolvedDir = conf.get("spark.eventLog.dir", EventLoggingListener.DEFAULT_LOG_DIR)
          .stripSuffix("/")
        Some(Utils.resolveURI(unresolvedDir))
      } else {
        None
      } //事件日志目录

    _eventLogCodec = {
      val compress = _conf.getBoolean("spark.eventLog.compress", false)
      if (compress && isEventLogEnabled) {
        Some(CompressionCodec.getCodecName(_conf)).map(CompressionCodec.getShortName)
      } else {
        None
      }
    } //事件日志压缩格式

    _listenerBus = new LiveListenerBus(_conf) //监听事件总线

    // Initialize the app status store and listener before SparkEnv is created so that it gets
    // all events.
    //在创建SparkEnv之前初始化应用程序状态存储和侦听器,以便它获取所有事件
    _statusStore = AppStatusStore.createLiveStore(conf) //初始化存储
    listenerBus.addToStatusQueue(_statusStore.listener.get) //加入总线

    // Create the Spark execution environment (cache, map output tracker, etc)
    //创建Spark执行环境(缓存,map输出,跟踪器等)
    _env = createSparkEnv(_conf, isLocal, listenerBus) //初始化SparkEnv
    SparkEnv.set(_env)

    // If running the REPL, register the repl's output dir with the file server.
    //如果运行REPL,请将repl的输出目录注册到文件服务器
    _conf.getOption("spark.repl.class.outputDir").foreach { path =>
      val replUri = _env.rpcEnv.fileServer.addDirectory("/classes", new File(path))
      _conf.set("spark.repl.class.uri", replUri)
    }

    _statusTracker = new SparkStatusTracker(this, _statusStore) //监控job和stage的低层次(low-level)状态的api

    _progressBar =
      if (_conf.get(UI_SHOW_CONSOLE_PROGRESS) && !log.isInfoEnabled) {
        Some(new ConsoleProgressBar(this))
      } else {
        None
      } //显示控制台下一行中stages的进度

    _ui =
      if (conf.getBoolean("spark.ui.enabled", true)) {
        Some(SparkUI.create(Some(this), _statusStore, _conf, _env.securityManager, appName, "",
          startTime))
      } else {
        // For tests, do not enable the UI
        None
      } //spark.ui初始化
    // Bind the UI before starting the task scheduler to communicate
    // the bound port to the cluster manager properly
    //在启动taskScheduler之前绑定UI,以便将绑定端口正确地传递给集群管理器(cluster manager)
    _ui.foreach(_.bind())

    _hadoopConfiguration = SparkHadoopUtil.get.newConfiguration(_conf) //读取hadoop配置

    // Add each JAR given through the constructor
    //添加通过构造函数给出的每个JAR
    if (jars != null) {
      jars.foreach(addJar)
    }

    if (files != null) {
      files.foreach(addFile)
    }

    _executorMemory = _conf.getOption("spark.executor.memory")
      .orElse(Option(System.getenv("SPARK_EXECUTOR_MEMORY")))
      .orElse(Option(System.getenv("SPARK_MEM"))
        .map(warnSparkMem))
      .map(Utils.memoryStringToMb)
      .getOrElse(1024) //executor内存,默认为1024m

    //executor配置
    // Convert java options to env vars as a work around
    // since we can't set env vars directly in sbt.
    //将java选项转换为env vars作为解决方法,因为我们无法直接在sbt中设置env vars
    for {(envKey, propKey) <- Seq(("SPARK_TESTING", "spark.testing"))
         value <- Option(System.getenv(envKey)).orElse(Option(System.getProperty(propKey)))} {
      executorEnvs(envKey) = value
    }
    Option(System.getenv("SPARK_PREPEND_CLASSES")).foreach { v =>
      executorEnvs("SPARK_PREPEND_CLASSES") = v
    }
    // The Mesos scheduler backend relies on this environment variable to set executor memory.
    //Mesos调度程序后端依赖此环境变量来设置执行程序内存
    // TODO: Set this only in the Mesos scheduler.
    executorEnvs("SPARK_EXECUTOR_MEMORY") = executorMemory + "m"
    executorEnvs ++= _conf.getExecutorEnv
    executorEnvs("SPARK_USER") = sparkUser

    // We need to register "HeartbeatReceiver" before "createTaskScheduler" because Executor will
    // retrieve "HeartbeatReceiver" in the constructor. (SPARK-6640)
    //我们需要在createTaskScheduler之前注册HeartbeatReceiver,因为Executor将在构造函数中检索HeartbeatReceiver (SPARK-6640)
    _heartbeatReceiver = env.rpcEnv.setupEndpoint(
      HeartbeatReceiver.ENDPOINT_NAME, new HeartbeatReceiver(this)) //注册心跳接收器(HeartbeatReceiver)

    //以下是SparkContext中最重要的部分,即创建一系列调度器(scheduler)
    // Create and start the scheduler
    //创建并启动scheduler
    val (sched, ts) = SparkContext.createTaskScheduler(this, master, deployMode)
    _schedulerBackend = sched //与master、worker通信收集worker上分配给该应用使用的资源情况
    _taskScheduler = ts //为Spark的任务调度资源分配器,spark通过他提交任务并请求集群调度任务
    _dagScheduler = new DAGScheduler(this) //为高级的、基于Stage的调度器,负责创建job,将DAG中的rdd划分到不同的stage,并将stage作为taskSets提交给底层调度器taskScheduler执行
    _heartbeatReceiver.ask[Boolean](TaskSchedulerIsSet) //查询taskScheduler是否创建

    // start TaskScheduler after taskScheduler sets DAGScheduler reference in DAGScheduler's
    // constructor
    //taskScheduler在DAGScheduler的构造函数中设置DAGScheduler引用后启动TaskScheduler
    _taskScheduler.start() //taskScheduler运行

    _applicationId = _taskScheduler.applicationId() //获取applicationId
    _applicationAttemptId = taskScheduler.applicationAttemptId() //获取applicationAttemptId
    _conf.set("spark.app.id", _applicationId)
    if (_conf.getBoolean("spark.ui.reverseProxy", false)) {
      System.setProperty("spark.ui.proxyBase", "/proxy/" + _applicationId)
    }
    _ui.foreach(_.setAppId(_applicationId))
    _env.blockManager.initialize(_applicationId) //blockManager初始化

    // The metrics system for Driver need to be set spark.app.id to app ID.
    // So it should start after we get app ID from the task scheduler and set spark.app.id.
    //Driver的metrics系统需要将spark.app.id设置为app ID,所以它应该从我们从任务调度程序获取app ID并设置spark.app.id后启动
    _env.metricsSystem.start() //metrics系统运行
    // Attach the driver metrics servlet handler to the web ui after the metrics system is started.
    //在metrics系统启动后,将driver metrics servlet handler附加到web ui
    _env.metricsSystem.getServletHandlers.foreach(handler => ui.foreach(_.attachHandler(handler)))

    _eventLogger =
      if (isEventLogEnabled) {
        val logger =
          new EventLoggingListener(_applicationId, _applicationAttemptId, _eventLogDir.get,
            _conf, _hadoopConfiguration)
        logger.start()
        listenerBus.addToEventLogQueue(logger)
        Some(logger)
      } else {
        None
      }

    // Optionally scale number of executors dynamically based on workload. Exposed for testing.
    //(可选)根据工作负载动态调整executor数量,暴露于测试
    val dynamicAllocationEnabled = Utils.isDynamicAllocationEnabled(_conf) //如果开启动态分配executor
    _executorAllocationManager =
      if (dynamicAllocationEnabled) {
        schedulerBackend match {
          case b: ExecutorAllocationClient =>
            Some(new ExecutorAllocationManager(
              schedulerBackend.asInstanceOf[ExecutorAllocationClient], listenerBus, _conf,
              _env.blockManager.master))
          case _ =>
            None
        }
      } else {
        None
      }
    _executorAllocationManager.foreach(_.start()) //遍历运行的executor

    _cleaner =
      if (_conf.getBoolean("spark.cleaner.referenceTracking", true)) {
        Some(new ContextCleaner(this))
      } else {
        None
      }
    _cleaner.foreach(_.start())

    setupAndStartListenerBus() //设置和启动事件总线
    postEnvironmentUpdate() //上传更新环境变量
    postApplicationStart() //上传application开始

    // Post init
    //初始化后
    _taskScheduler.postStartHook()
    _env.metricsSystem.registerSource(_dagScheduler.metricsSource)
    _env.metricsSystem.registerSource(new BlockManagerSource(_env.blockManager))
    _executorAllocationManager.foreach { e =>
      _env.metricsSystem.registerSource(e.executorAllocationManagerSource)
    }

    // Make sure the context is stopped if the user forgets about it. This avoids leaving
    // unfinished event logs around after the JVM exits cleanly. It doesn't help if the JVM
    // is killed, though.
    //如果用户忘记了context,请确保context已停止.这样可以避免在JVM彻底退出后留下未完成的事件日志.但是,如果JVM被杀,它无济于事
    logDebug("Adding shutdown hook") // force eager creation of logger (迫切渴望创建记录器)
    _shutdownHookRef = ShutdownHookManager.addShutdownHook(
      ShutdownHookManager.SPARK_CONTEXT_SHUTDOWN_PRIORITY) { () =>
      logInfo("Invoking stop() from shutdown hook")
      try {
        stop()
      } catch {
        case e: Throwable =>
          logWarning("Ignoring Exception while stopping SparkContext from shutdown hook", e)
      }
    }
  } catch {
    case NonFatal(e) =>
      logError("Error initializing SparkContext.", e)
      try {
        stop()
      } catch {
        case NonFatal(inner) =>
          logError("Error stopping SparkContext after init error.", inner)
      } finally {
        throw e
      }
  }
```

其中SparkContext重要对象有:
- SparkEnv
- DAGScheduler
- TaskScheduler
- SchedulerBackend

接下的文章继续阅读分析这些对象



