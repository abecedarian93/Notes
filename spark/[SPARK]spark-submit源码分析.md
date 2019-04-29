### Spark-submit源码分析

###### 已经有很多关于spark-submit的源码阅读和分析了,我们从spark-on-yarn的角度从spark-submit作业提交到运行的路径删繁就简,给出最简洁的分析版
> spark版本:2.4

##### 1.当我们编写一个程序
````
package com.abecedarian.demo.spark

import org.apache.spark.{SparkConf, SparkContext}

object SparkHelloWord extends App {

  override def main(args: Array[String]): Unit = {

    val conf = new SparkConf()
    conf.setAppName("SparkHelloWorld")
    val sc = new SparkContext(conf)

    val lines = sc.textFile("data/spark/hellowork.txt")
    val wc = lines.flatMap(_.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
    wc.foreach(println)


    Thread.sleep(10 * 60 * 1000) // 挂住 10 分钟;
  }
}
````
> 提交这个程序到集群上运行,使用$SPARK_HOME/bin目录下的spark-submit 脚本去提交用户的程序

##### 2.提交作业到yarn
````
./bin/spark-submit \
  --class com.abecedarian.demo.spark.SparkHelloWord \
  --master yarn \
  --deploy-mode client \
  --executor-memory 2g \
  --num-executors 2 \
  --executor-cores 1 \
  demo-tutorial-1.0.0-jar-with-dependencies.jar
````

> 查看spark-submit脚本,发现最终调用的是**org.apache.spark.deploy.SparkSubmit**类
````
exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
````
> org.apache.spark.deploy.SparkSubmit的main方法
````
override def main(args: Array[String]): Unit = {
    val submit = new SparkSubmit() {
      self =>
      
      //解析参数
      override protected def parseArguments(args: Array[String]): SparkSubmitArguments = {
        new SparkSubmitArguments(args) {
          override protected def logInfo(msg: => String): Unit = self.logInfo(msg)

          override protected def logWarning(msg: => String): Unit = self.logWarning(msg)
        }
      }

      //打印日志
      override protected def logInfo(msg: => String): Unit = printMessage(msg)

      //警告日志
      override protected def logWarning(msg: => String): Unit = printMessage(s"Warning: $msg")

      //提交作业
      override def doSubmit(args: Array[String]): Unit = {
        try {
          super.doSubmit(args)
        } catch {
          case e: SparkUserAppException =>
            exitFn(e.exitCode)
        }
      }

    }

    submit.doSubmit(args)
  }
````

> 进入doSubmit函数
````
def doSubmit(args: Array[String]): Unit = {
    // Initialize logging if it hasn't been done yet. Keep track of whether logging needs to
    // be reset before the application starts.
    val uninitLog = initializeLogIfNecessary(true, silent = true)

    val appArgs = parseArguments(args) //解析参数
    if (appArgs.verbose) {
      logInfo(appArgs.toString)
    }
    appArgs.action match {
      case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog)  //提交作业
      case SparkSubmitAction.KILL => kill(appArgs) //杀死作业
      case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs) //查询状态
      case SparkSubmitAction.PRINT_VERSION => printVersion() //打印版本号
    }
  }
````

> 进入submit函数
````
private def submit(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {
    val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)

    def doRunMain(): Unit = {
      if (args.proxyUser != null) { //使用代理用户
        val proxyUser = UserGroupInformation.createProxyUser(args.proxyUser,
          UserGroupInformation.getCurrentUser())
        try {
          proxyUser.doAs(new PrivilegedExceptionAction[Unit]() {
            override def run(): Unit = {
              runMain(childArgs, childClasspath, sparkConf, childMainClass, args.verbose)
            }
          })
        } catch {
          case e: Exception =>
            // Hadoop's AuthorizationException suppresses the exception's stack trace, which
            // makes the message printed to the output by the JVM not very helpful. Instead,
            // detect exceptions with empty stack traces here, and treat them differently.
            if (e.getStackTrace().length == 0) {
              error(s"ERROR: ${e.getClass().getName()}: ${e.getMessage()}")
            } else {
              throw e
            }
        }
      } else {
        runMain(childArgs, childClasspath, sparkConf, childMainClass, args.verbose) //我们关心的主入口
      }
    }

    // Let the main class re-initialize the logging system once it starts.
    if (uninitLog) {
      Logging.uninitialize()
    }

    // In standalone cluster mode, there are two submission gateways:
    //   (1) The traditional RPC gateway using o.a.s.deploy.Client as a wrapper
    //   (2) The new REST-based gateway introduced in Spark 1.3
    // The latter is the default behavior as of Spark 1.3, but Spark submit will fail over
    // to use the legacy gateway if the master endpoint turns out to be not a REST server.
    if (args.isStandaloneCluster && args.useRest) { //standalone-cluster and rest模式
      try {
        logInfo("Running Spark using the REST application submission protocol.")
        doRunMain()
      } catch {
        // Fail over to use the legacy submission gateway
        case e: SubmitRestConnectionException =>
          logWarning(s"Master endpoint ${args.master} was not a REST server. " +
            "Falling back to legacy submission gateway instead.")
          args.useRest = false
          submit(args, false)
      }
    // In all other modes, just run the main class as prepared
    } else {
      doRunMain()
    }
  }
````

> 进入runMain函数
````
private def runMain(
      childArgs: Seq[String],
      childClasspath: Seq[String],
      sparkConf: SparkConf,
      childMainClass: String,
      verbose: Boolean): Unit = {
    if (verbose) {  //使用verbose参数打印更多信息
      logInfo(s"Main class:\n$childMainClass")
      logInfo(s"Arguments:\n${childArgs.mkString("\n")}")
      // sysProps may contain sensitive information, so redact before printing
      logInfo(s"Spark config:\n${Utils.redact(sparkConf.getAll.toMap).mkString("\n")}")
      logInfo(s"Classpath elements:\n${childClasspath.mkString("\n")}")
      logInfo("\n")
    }

    val loader =
      if (sparkConf.get(DRIVER_USER_CLASS_PATH_FIRST)) {  //设置spark.driver.userClassPathFirst=true,先加载用户定义的类
        new ChildFirstURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      } else {
        new MutableURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      }
    Thread.currentThread.setContextClassLoader(loader)

    for (jar <- childClasspath) {
      addJarToClasspath(jar, loader)
    }

    var mainClass: Class[_] = null

    try {
      mainClass = Utils.classForName(childMainClass) //JVM查找并加载指定的类,JVM会执行该类的静态代码段
    } catch {
      case e: ClassNotFoundException =>
        logWarning(s"Failed to load $childMainClass.", e)
        if (childMainClass.contains("thriftserver")) {
          logInfo(s"Failed to load main class $childMainClass.")
          logInfo("You need to build Spark with -Phive and -Phive-thriftserver.")
        }
        throw new SparkUserAppException(CLASS_NOT_FOUND_EXIT_STATUS)
      case e: NoClassDefFoundError =>
        logWarning(s"Failed to load $childMainClass: ${e.getMessage()}")
        if (e.getMessage.contains("org/apache/hadoop/hive")) {
          logInfo(s"Failed to load hive class.")
          logInfo("You need to build Spark with -Phive and -Phive-thriftserver.")
        }
        throw new SparkUserAppException(CLASS_NOT_FOUND_EXIT_STATUS)
    }

   //spark作业的入口点，包装了主类
    val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
      mainClass.newInstance().asInstanceOf[SparkApplication]
    } else {
      // SPARK-4170
      if (classOf[scala.App].isAssignableFrom(mainClass)) {
        logWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
      }
      new JavaMainApplication(mainClass)
    }

    @tailrec
    def findCause(t: Throwable): Throwable = t match {
      case e: UndeclaredThrowableException =>
        if (e.getCause() != null) findCause(e.getCause()) else e
      case e: InvocationTargetException =>
        if (e.getCause() != null) findCause(e.getCause()) else e
      case e: Throwable =>
        e
    }

    try {
      app.start(childArgs.toArray, sparkConf) //spark作业开始
    } catch {
      case t: Throwable =>
        throw findCause(t)
    }
  }

  /** Throw a SparkException with the given error message. */
  private def error(msg: String): Unit = throw new SparkException(msg)

}
````

> 进入SparkApplication.start()函数
````
override def start(args: Array[String], conf: SparkConf): Unit = {
    val mainMethod = klass.getMethod("main", new Array[String](0).getClass)
    if (!Modifier.isStatic(mainMethod.getModifiers)) {
      throw new IllegalStateException("The main method in the given main class must be static")
    }

    val sysProps = conf.getAll.toMap
    sysProps.foreach { case (k, v) =>
      sys.props(k) = v
    }

    mainMethod.invoke(null, args) //通过反射mainMethod.invoke执行该方法 com.abecedarian.demo.spark.SparkHelloWord #main()
  }
````

> SparkContext 源码分析

> SparkContext变量:
````
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

> SparkContext初始化:
````
_conf = config.clone()
    _conf.validateSettings()

    if (!_conf.contains("spark.master")) {
      throw new SparkException("A master URL must be set in your configuration")
    }
    if (!_conf.contains("spark.app.name")) {
      throw new SparkException("An application name must be set in your configuration")
    }

    // log out spark.app.name in the Spark driver logs
    logInfo(s"Submitted application: $appName")

    // System property spark.yarn.app.id must be set if user code ran by AM on a YARN cluster
    if (master == "yarn" && deployMode == "cluster" && !_conf.contains("spark.yarn.app.id")) {
      throw new SparkException("Detected yarn cluster mode, but isn't running on a cluster. " +
        "Deployment to YARN is not supported directly by SparkContext. Please use spark-submit.")
    }

    if (_conf.getBoolean("spark.logConf", false)) {
      logInfo("Spark configuration:\n" + _conf.toDebugString)
    }

    // Set Spark driver host and port system properties. This explicitly sets the configuration
    // instead of relying on the default value of the config constant.
    _conf.set(DRIVER_HOST_ADDRESS, _conf.get(DRIVER_HOST_ADDRESS))
    _conf.setIfMissing("spark.driver.port", "0")

    _conf.set("spark.executor.id", SparkContext.DRIVER_IDENTIFIER)

    _jars = Utils.getUserJars(_conf)  //获取jars,在yarn运行模式下返回空,yarn有自己获取jars的机制
    _files = _conf.getOption("spark.files").map(_.split(",")).map(_.filter(_.nonEmpty))
      .toSeq.flatten  //获取files

    _eventLogDir =
      if (isEventLogEnabled) {
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
    }  //事件日志压缩格式

    _listenerBus = new LiveListenerBus(_conf)  //监听事件总线

    // Initialize the app status store and listener before SparkEnv is created so that it gets
    // all events.
    _statusStore = AppStatusStore.createLiveStore(conf) //初始化存储
    listenerBus.addToStatusQueue(_statusStore.listener.get) //加入总线

    // Create the Spark execution environment (cache, map output tracker, etc)
    _env = createSparkEnv(_conf, isLocal, listenerBus) //初始化SparkEnv
    SparkEnv.set(_env)

    // If running the REPL, register the repl's output dir with the file server.
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
    _ui.foreach(_.bind()) //taskScheduler运行之前绑定ui

    _hadoopConfiguration = SparkHadoopUtil.get.newConfiguration(_conf) //读取hadoop配置

    // Add each JAR given through the constructor
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
    for { (envKey, propKey) <- Seq(("SPARK_TESTING", "spark.testing"))
      value <- Option(System.getenv(envKey)).orElse(Option(System.getProperty(propKey)))} {
      executorEnvs(envKey) = value
    }
    Option(System.getenv("SPARK_PREPEND_CLASSES")).foreach { v =>
      executorEnvs("SPARK_PREPEND_CLASSES") = v
    }
    // The Mesos scheduler backend relies on this environment variable to set executor memory.
    // TODO: Set this only in the Mesos scheduler.
    executorEnvs("SPARK_EXECUTOR_MEMORY") = executorMemory + "m"
    executorEnvs ++= _conf.getExecutorEnv
    executorEnvs("SPARK_USER") = sparkUser

    // We need to register "HeartbeatReceiver" before "createTaskScheduler" because Executor will
    // retrieve "HeartbeatReceiver" in the constructor. (SPARK-6640)
    _heartbeatReceiver = env.rpcEnv.setupEndpoint(
      HeartbeatReceiver.ENDPOINT_NAME, new HeartbeatReceiver(this)) //创建taskScheduler之前注册心跳接收器

    // Create and start the scheduler
    val (sched, ts) = SparkContext.createTaskScheduler(this, master, deployMode)
    _schedulerBackend = sched //与master、worker通信收集worker上分配给该应用使用的资源情况
    _taskScheduler = ts //为Spark的任务调度资源分配器,spark通过他提交任务并请求集群调度任务
    _dagScheduler = new DAGScheduler(this) //为高级的、基于Stage的调度器,负责创建job,将DAG中的rdd划分到不同的stage,并将stage作为taskSets提交给底层调度器taskScheduler执行
    _heartbeatReceiver.ask[Boolean](TaskSchedulerIsSet) //查询taskScheduler是否创建

    // start TaskScheduler after taskScheduler sets DAGScheduler reference in DAGScheduler's
    // constructor
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
    _env.metricsSystem.start() //metricsSystem运行
    // Attach the driver metrics servlet handler to the web ui after the metrics system is started.
    _env.metricsSystem.getServletHandlers.foreach(handler => ui.foreach(_.attachHandler(handler))) //将driver metrics servlet handler传递给web ui

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
    _taskScheduler.postStartHook()
    _env.metricsSystem.registerSource(_dagScheduler.metricsSource)
    _env.metricsSystem.registerSource(new BlockManagerSource(_env.blockManager))
    _executorAllocationManager.foreach { e =>
      _env.metricsSystem.registerSource(e.executorAllocationManagerSource)
    }

    // Make sure the context is stopped if the user forgets about it. This avoids leaving
    // unfinished event logs around after the JVM exits cleanly. It doesn't help if the JVM
    // is killed, though.
    logDebug("Adding shutdown hook") // force eager creation of logger
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
````