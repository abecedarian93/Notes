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