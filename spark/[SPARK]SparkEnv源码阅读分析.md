# SparkEnv源码阅读分析

### SparkEnv:

保存正在运行的Spark实例(master或worker)的所有运行时环境对象,包括序列化程序,RpcEnv,块管理器(block-manager)，map输出跟踪器等

当前Spark代码通过全局变量查找SparkEnv,因此所有线程都可以访问相同的SparkEnv.它可以通过SparkEnv.get访问(例如,在创建SparkContext之后)

注意:这不适合外部使用.这是针对Shark的,并且可能在将来的版本中变为私有

SparkEnv主要创建的对象:
- securityManager (安全管理器)
- rpcEnv (rpc运行环境)
- serializerManager (序列化管理器)
- broadcastManager (广播变量管理器)
- mapOutputTracker (map输出跟踪器)
- shuffleManager (shuffle管理)
- memoryManager (内存管理)
- blockManager (块管理)
- metricsSystem (统计系统)
- outputCommitCoordinator (输出提交控制器)

SparkContext中初始化SparkEnv代码:
````
 _env = createSparkEnv(_conf, isLocal, listenerBus)
 
  private[spark] def createSparkEnv(
                                     conf: SparkConf,
                                     isLocal: Boolean,
                                     listenerBus: LiveListenerBus): SparkEnv = {
    SparkEnv.createDriverEnv(conf, isLocal, listenerBus, SparkContext.numDriverCores(master, conf))
  }
````
最终调用`SparkEnv.createDriverEnv`方法进行初始化

CoarseGrainedExecutorBackend中初始化SparkEnv代码:
````
 val env = SparkEnv.createExecutorEnv(
        driverConf, executorId, hostname, cores, cfg.ioEncryptionKey, isLocal = false)
````
最终调用`SparkEnv.createExecutorEnv`方法进行初始化

### SparkEnv初始化:

SparkEnv初始化主要分为`createDriverEnv`和`createExecutorEnv`,分别代表创建driver和executor的运行环境,它们最终都调用了`create`方法

#### createDriverEnv:
````
//为driver创建SparkEnv
private[spark] def createDriverEnv(
     conf: SparkConf,
     isLocal: Boolean,
     listenerBus: LiveListenerBus,
     numCores: Int,
     mockOutputCommitCoordinator: Option[OutputCommitCoordinator] = None): SparkEnv = {
   assert(conf.contains(DRIVER_HOST_ADDRESS),
     s"${DRIVER_HOST_ADDRESS.key} is not set on the driver!")
   assert(conf.contains("spark.driver.port"), "spark.driver.port is not set on the driver!")
   val bindAddress = conf.get(DRIVER_BIND_ADDRESS)
   val advertiseAddress = conf.get(DRIVER_HOST_ADDRESS)
   val port = conf.get("spark.driver.port").toInt
   val ioEncryptionKey = if (conf.get(IO_ENCRYPTION_ENABLED)) {
     Some(CryptoStreamUtils.createKey(conf))
   } else {
     None
   }
   create(
     conf,
     SparkContext.DRIVER_IDENTIFIER,
     bindAddress,
     advertiseAddress,
     Option(port),
     isLocal,
     numCores,
     ioEncryptionKey,
     listenerBus = listenerBus,
     mockOutputCommitCoordinator = mockOutputCommitCoordinator
   )
 }
````

#### createExecutorEnv:
````
//为executor创建SparkEnv
//在coarse-grained模式中,executor提供已经实例化的RpcEnv
private[spark] def createExecutorEnv(
      conf: SparkConf,
      executorId: String,
      hostname: String,
      numCores: Int,
      ioEncryptionKey: Option[Array[Byte]],
      isLocal: Boolean): SparkEnv = {
    val env = create(
      conf,
      executorId,
      hostname,
      hostname,
      None,
      isLocal,
      numCores,
      ioEncryptionKey
    )
    SparkEnv.set(env)
    env
  }
````

#### create:
````
//为driver或executor创建SparkEnv的辅助方法
  private def create(
      conf: SparkConf,
      executorId: String,
      bindAddress: String,
      advertiseAddress: String,
      port: Option[Int],
      isLocal: Boolean,
      numUsableCores: Int,
      ioEncryptionKey: Option[Array[Byte]],
      listenerBus: LiveListenerBus = null,
      mockOutputCommitCoordinator: Option[OutputCommitCoordinator] = None): SparkEnv = {

    val isDriver = executorId == SparkContext.DRIVER_IDENTIFIER //通过DRIVER_IDENTIFIER和executorId判断是否为driver调用

    // Listener bus is only used on the driver
    //监听器总线仅用于driver
    //监听器(listenerBus)不能为null
    if (isDriver) {
      assert(listenerBus != null, "Attempted to create driver SparkEnv with null listener bus!")
    }

    //实例化securityManager
    val securityManager = new SecurityManager(conf, ioEncryptionKey)
    if (isDriver) {
      securityManager.initializeAuth()
    }

    ioEncryptionKey.foreach { _ =>
      if (!securityManager.isEncryptionEnabled()) {
        logWarning("I/O encryption enabled without RPC encryption: keys will be visible on the " +
          "wire.")
      }
    }

    val systemName = if (isDriver) driverSystemName else executorSystemName
    //创建rpcEnv
    val rpcEnv = RpcEnv.create(systemName, bindAddress, advertiseAddress, port.getOrElse(-1), conf,
      securityManager, numUsableCores, !isDriver)

    // Figure out which port RpcEnv actually bound to in case the original port is 0 or occupied.
    //确定在原始端口为0或已占用的情况下,RpcEnv实际绑定到哪个端口
    if (isDriver) {
      conf.set("spark.driver.port", rpcEnv.address.port.toString)
    }

    // Create an instance of the class with the given name, possibly initializing it with our conf
    //创建具有给定名称的类的实例,可能使用我们的conf初始化它
    def instantiateClass[T](className: String): T = {
      val cls = Utils.classForName(className)
      // Look for a constructor taking a SparkConf and a boolean isDriver, then one taking just
      // SparkConf, then one taking no arguments
      //寻找一个构造函数接受一个SparkConf和一个布尔值isDriver,然后一个只接受SparkConf,然后一个不带参数
      try {
        cls.getConstructor(classOf[SparkConf], java.lang.Boolean.TYPE)
          .newInstance(conf, new java.lang.Boolean(isDriver))
          .asInstanceOf[T]
      } catch {
        case _: NoSuchMethodException =>
          try {
            cls.getConstructor(classOf[SparkConf]).newInstance(conf).asInstanceOf[T]
          } catch {
            case _: NoSuchMethodException =>
              cls.getConstructor().newInstance().asInstanceOf[T]
          }
      }
    }

    // Create an instance of the class named by the given SparkConf property, or defaultClassName
    // if the property is not set, possibly initializing it with our conf
    //创建由给定SparkConf属性或defaultClassName命名的类的实例,如果未设置属性,可能使用我们的conf初始化它
    def instantiateClassFromConf[T](propertyName: String, defaultClassName: String): T = {
      instantiateClass[T](conf.get(propertyName, defaultClassName))
    }
    //设置序列化器,可以看到默认使用的是Java的序列化器
    val serializer = instantiateClassFromConf[Serializer](
      "spark.serializer", "org.apache.spark.serializer.JavaSerializer")
    logDebug(s"Using serializer: ${serializer.getClass}")

    val serializerManager = new SerializerManager(serializer, conf, ioEncryptionKey)

    val closureSerializer = new JavaSerializer(conf)

    def registerOrLookupEndpoint(
        name: String, endpointCreator: => RpcEndpoint):
      RpcEndpointRef = {
      if (isDriver) {
        logInfo("Registering " + name)
        rpcEnv.setupEndpoint(name, endpointCreator)
      } else {
        RpcUtils.makeDriverRef(name, conf, rpcEnv)
      }
    }

    //广播变量管理器
    val broadcastManager = new BroadcastManager(isDriver, conf, securityManager)

    //如果是Driver实例化MapOutputTrackerMaster,如果是Executor实例化MapOutputTrackerWorker
    //说明MapOutputTracker也是Master／Slaves的结构
    val mapOutputTracker = if (isDriver) {
      new MapOutputTrackerMaster(conf, broadcastManager, isLocal)
    } else {
      new MapOutputTrackerWorker(conf)
    }

    // Have to assign trackerEndpoint after initialization as MapOutputTrackerEndpoint
    // requires the MapOutputTracker itself
    //由于MapOutputTrackerEndpoint需要MapOutputTracker本身,因此必须在初始化后分配trackerEndpoint
    mapOutputTracker.trackerEndpoint = registerOrLookupEndpoint(MapOutputTracker.ENDPOINT_NAME,
      new MapOutputTrackerMasterEndpoint(
        rpcEnv, mapOutputTracker.asInstanceOf[MapOutputTrackerMaster], conf))

    // Let the user specify short names for shuffle managers
    //让用户为shuffle-managers指定短名称
    //Shuffle的配置信息,使用的是SortShuffleManager,已取消HashShuffleManager
    val shortShuffleMgrNames = Map(
      "sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName,
      "tungsten-sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName)
    val shuffleMgrName = conf.get("spark.shuffle.manager", "sort")
    val shuffleMgrClass =
      shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase(Locale.ROOT), shuffleMgrName)
    //实例化ShuffleManager,默认是SortShuffleManager
    val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)

    //是否使用原始的MemoryManager,即StaticMemoryManager,默认为false
    val useLegacyMemoryManager = conf.getBoolean("spark.memory.useLegacyMode", false)
    val memoryManager: MemoryManager =
      if (useLegacyMemoryManager) {
        new StaticMemoryManager(conf, numUsableCores) //静态内存管理
      } else {
        UnifiedMemoryManager(conf, numUsableCores)  //统一内存管理,默认
      }

    //根据driver或executor获取blockManager端口
    val blockManagerPort = if (isDriver) {
      conf.get(DRIVER_BLOCK_MANAGER_PORT)
    } else {
      conf.get(BLOCK_MANAGER_PORT)
    }

    //实例化blockTransferService,默认的实现方式是Netty,即NettyBlockTransferService
    val blockTransferService =
      new NettyBlockTransferService(conf, securityManager, bindAddress, advertiseAddress,
        blockManagerPort, numUsableCores)
    //blockManager相关的初始化过程
    val blockManagerMaster = new BlockManagerMaster(registerOrLookupEndpoint(
      BlockManagerMaster.DRIVER_ENDPOINT_NAME,
      new BlockManagerMasterEndpoint(rpcEnv, isLocal, conf, listenerBus)),
      conf, isDriver)

    // NB: blockManager is not valid until initialize() is called later.
    //注意：在稍后调用initialize()之前,blockManager无效
    val blockManager = new BlockManager(executorId, rpcEnv, blockManagerMaster,
      serializerManager, conf, memoryManager, mapOutputTracker, shuffleManager,
      blockTransferService, securityManager, numUsableCores)

    //统计系统(metrics系统)
    val metricsSystem = if (isDriver) {
      // Don't start metrics system right now for Driver.
      // We need to wait for the task scheduler to give us an app ID.
      // Then we can start the metrics system.
      //现在不要为driver启动metrics系统,我们需要等待taskScheduler为我们提供appID,然后我们可以启动metrics系统
      MetricsSystem.createMetricsSystem("driver", conf, securityManager)
    } else {
      // We need to set the executor ID before the MetricsSystem is created because sources and
      // sinks specified in the metrics configuration file will want to incorporate this executor's
      // ID into the metrics they report.
      //我们需要在创建MetricsSystem之前设置executor ID,
      //因为metrics配置文件中指定的源(sources)和接收器(sinks)会希望将此executor的ID合并到它们报告的metrics中
      conf.set("spark.executor.id", executorId)
      val ms = MetricsSystem.createMetricsSystem("executor", conf, securityManager)
      ms.start()
      ms
    }

    //outputCommitCoordinator相关的初始化及注册部分
    val outputCommitCoordinator = mockOutputCommitCoordinator.getOrElse {
      new OutputCommitCoordinator(conf, isDriver)
    }
    val outputCommitCoordinatorRef = registerOrLookupEndpoint("OutputCommitCoordinator",
      new OutputCommitCoordinatorEndpoint(rpcEnv, outputCommitCoordinator))
    outputCommitCoordinator.coordinatorRef = Some(outputCommitCoordinatorRef)

    val envInstance = new SparkEnv(
      executorId,
      rpcEnv,
      serializer,
      closureSerializer,
      serializerManager,
      mapOutputTracker,
      shuffleManager,
      broadcastManager,
      blockManager,
      securityManager,
      metricsSystem,
      memoryManager,
      outputCommitCoordinator,
      conf)

    // Add a reference to tmp dir created by driver, we will delete this tmp dir when stop() is
    // called, and we only need to do it for driver. Because driver may run as a service, and if we
    // don't delete this tmp dir when sc is stopped, then will create too many tmp dirs.
    //添加对driver创建的tmp目录的引用,我们将在调用stop()时删除此tmp目录,我们只需要为driver执行此操作
    //因为driver可以作为服务运行,并且如果我们在sc停止时不删除此tmp目录,那么将创建太多的tmp目录
    //sparkFiles的存储目录,用来下载依赖
    if (isDriver) {
      val sparkFilesDir = Utils.createTempDir(Utils.getLocalDir(conf), "userFiles").getAbsolutePath
      envInstance.driverTmpDir = Some(sparkFilesDir)
    }

    envInstance
  }
````
