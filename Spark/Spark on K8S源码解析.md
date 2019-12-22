---
title: Spark on K8S源码解析 
tags: spark,k8s
grammar_cjkRuby: true
grammar_highlight: true
grammar_codeLinenums: true
grammar_code: true
---
time: `2019-12-19 `
[TOC]
# Spark on k8s源码解析
本文基于spark-3.0.0 preview源码,来分析spark作业基于K8S的提交过程.
spark on k8s的代码位置位于:
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1576754537441.png)
关于kubernetes目录由以下部分组成:

``` bash
$ tree kubernetes -L 1 
kubernetes
├── >>==core==<<
├── >>==docker==<<
└── integration-tests

```
其中kubernetes中的core/src/main的代码目录如下:

``` bash
$ tree core/src/main/scala -L 7
core/src/main/scala
└── org
    └── apache
        └── spark
            ├── deploy
            │   └── k8s
            │       ├── Config.scala
            │       ├── Constants.scala
            │       ├── features
            │       │   ├── BasicDriverFeatureStep.scala
            │       │   ├── BasicExecutorFeatureStep.scala
            │       │   ├── DriverCommandFeatureStep.scala
            │       │   ├── DriverKubernetesCredentialsFeatureStep.scala
            │       │   ├── DriverServiceFeatureStep.scala
            │       │   ├── EnvSecretsFeatureStep.scala
            │       │   ├── ExecutorKubernetesCredentialsFeatureStep.scala
            │       │   ├── HadoopConfDriverFeatureStep.scala
            │       │   ├── KerberosConfDriverFeatureStep.scala
            │       │   ├── KubernetesFeatureConfigStep.scala
            │       │   ├── LocalDirsFeatureStep.scala
            │       │   ├── MountSecretsFeatureStep.scala
            │       │   ├── MountVolumesFeatureStep.scala
            │       │   └── PodTemplateConfigMapStep.scala
            │       ├── KubernetesConf.scala
            │       ├── KubernetesDriverSpec.scala
            │       ├── KubernetesUtils.scala
            │       ├── KubernetesVolumeSpec.scala
            │       ├── KubernetesVolumeUtils.scala
            │       ├── SparkKubernetesClientFactory.scala
            │       ├── SparkPod.scala
            │       └── submit
            │           ├── K8sSubmitOps.scala
            │           ├── KubernetesClientApplication.scala
            │           ├── KubernetesDriverBuilder.scala
            │           ├── LoggingPodStatusWatcher.scala
            │           └── MainAppResource.scala
            └── scheduler
                └── cluster
                    └── k8s
                        ├── ExecutorPodsAllocator.scala
                        ├── ExecutorPodsLifecycleManager.scala
                        ├── ExecutorPodsPollingSnapshotSource.scala
                        ├── ExecutorPodsSnapshot.scala
                        ├── ExecutorPodsSnapshotsStoreImpl.scala
                        ├── ExecutorPodsSnapshotsStore.scala
                        ├── ExecutorPodStates.scala
                        ├── ExecutorPodsWatchSnapshotSource.scala
                        ├── KubernetesClusterManager.scala
                        ├── KubernetesClusterSchedulerBackend.scala
                        └── KubernetesExecutorBuilder.scala

10 directories, 39 files
```
而docker目录下面则是用来打包Spark镜像的Dockerfile:

``` bash
$ tree kubernetes/docker/src/main -L 5
kubernetes/docker/src/main
└── dockerfiles
    └── spark
        ├── bindings
        │   ├── python
        │   │   └── Dockerfile
        │   └── R
        │       └── Dockerfile
        ├── Dockerfile
        └── entrypoint.sh

5 directories, 4 files

```

## 1. Spark Submit
首先我们提交一个spark-pi的例子作为开始:

``` bash
$ ./bin/spark-submit \
    --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
    --deploy-mode cluster \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.executor.instances=5 \
    --conf spark.kubernetes.container.image=<spark-image> \
    local:///path/to/examples.jar
```
### spark-submit.sh

```shell?linenums&fancy=7
if [ -z "${SPARK_HOME}" ]; then
  source "$(dirname "$0")"/find-spark-home
fi

# disable randomized hash for string in Python 3.3+
export PYTHONHASHSEED=0
>>==# 源码批注: 这里将spark-submit中的所有入参都传递给spark-class==<< 
exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```
### spark-class.sh
这个脚本中核心功能见该代码的71-74行.
主要功能:
根据环境和spark-submit的入参去拼接
```bash
java -Xmx128m *** -cp *** com.demo.Main ***.jar
```
而进入的Main就是`org.apache.spark.deploy.SparkSubmit`
``` shell?linenums&fancy=71,72,73,74
#!/usr/bin/env bash

#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

if [ -z "${SPARK_HOME}" ]; then
  source "$(dirname "$0")"/find-spark-home
fi

. "${SPARK_HOME}"/bin/load-spark-env.sh

# Find the java binary
if [ -n "${JAVA_HOME}" ]; then
  RUNNER="${JAVA_HOME}/bin/java"
else
  if [ "$(command -v java)" ]; then
    RUNNER="java"
  else
    echo "JAVA_HOME is not set" >&2
    exit 1
  fi
fi

# Find Spark jars.
if [ -d "${SPARK_HOME}/jars" ]; then
  SPARK_JARS_DIR="${SPARK_HOME}/jars"
else
  SPARK_JARS_DIR="${SPARK_HOME}/assembly/target/scala-$SPARK_SCALA_VERSION/jars"
fi

if [ ! -d "$SPARK_JARS_DIR" ] && [ -z "$SPARK_TESTING$SPARK_SQL_TESTING" ]; then
  echo "Failed to find Spark jars directory ($SPARK_JARS_DIR)." 1>&2
  echo "You need to build Spark with the target \"package\" before running this program." 1>&2
  exit 1
else
  LAUNCH_CLASSPATH="$SPARK_JARS_DIR/*"
fi

# Add the launcher build dir to the classpath if requested.
if [ -n "$SPARK_PREPEND_CLASSES" ]; then
  LAUNCH_CLASSPATH="${SPARK_HOME}/launcher/target/scala-$SPARK_SCALA_VERSION/classes:$LAUNCH_CLASSPATH"
fi

# For tests
if [[ -n "$SPARK_TESTING" ]]; then
  unset YARN_CONF_DIR
  unset HADOOP_CONF_DIR
fi

# The launcher library will print arguments separated by a NULL character, to allow arguments with
# characters that would be otherwise interpreted by the shell. Read that in a while loop, populating
# an array that will be used to exec the final command.
#
# The exit code of the launcher is appended to the output, so the parent shell removes it from the
# command array and checks the value to see if the launcher succeeded.
>>==
build_command() {
  "$RUNNER" -Xmx128m $SPARK_LAUNCHER_OPTS -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main "$@"
  printf "%d\0" $?
}
==<<
# Turn off posix mode since it does not allow process substitution
set +o posix
CMD=()
DELIM=$'\n'
CMD_START_FLAG="false"
while IFS= read -d "$DELIM" -r ARG; do
  if [ "$CMD_START_FLAG" == "true" ]; then
    CMD+=("$ARG")
  else
    if [ "$ARG" == $'\0' ]; then
      # After NULL character is consumed, change the delimiter and consume command string.
      DELIM=''
      CMD_START_FLAG="true"
    elif [ "$ARG" != "" ]; then
      echo "$ARG"
    fi
  fi
done < <(build_command "$@")

COUNT=${#CMD[@]}
LAST=$((COUNT - 1))
LAUNCHER_EXIT_CODE=${CMD[$LAST]}

# Certain JVM failures result in errors being printed to stdout (instead of stderr), which causes
# the code that parses the output of the launcher to get confused. In those cases, check if the
# exit code is an integer, and if it's not, handle it as a special error case.
if ! [[ $LAUNCHER_EXIT_CODE =~ ^[0-9]+$ ]]; then
  echo "${CMD[@]}" | head -n-1 1>&2
  exit 1
fi

if [ $LAUNCHER_EXIT_CODE != 0 ]; then
  exit $LAUNCHER_EXIT_CODE
fi

CMD=("${CMD[@]:0:$LAST}")
exec "${CMD[@]}"

```
### SparkSubmit
通过java命令启动,首先进入 Object SparkSubmit的main方法:
- 构造SparkSubmit对象
- 执行SparkSubmit.doSubmit方法.

``` scala
  override def main(args: Array[String]): Unit = {
    val submit = new SparkSubmit() {
      self =>

      override protected def parseArguments(args: Array[String]): SparkSubmitArguments = {
        new SparkSubmitArguments(args) {
          override protected def logInfo(msg: => String): Unit = self.logInfo(msg)

          override protected def logWarning(msg: => String): Unit = self.logWarning(msg)

          override protected def logError(msg: => String): Unit = self.logError(msg)
        }
      }

      override protected def logInfo(msg: => String): Unit = printMessage(msg)

      override protected def logWarning(msg: => String): Unit = printMessage(s"Warning: $msg")

      override protected def logError(msg: => String): Unit = printMessage(s"Error: $msg")

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
```
这里我们进入SparkSubmit的doSubmit方法:
这里执行两步:
- parseArguments(args)构造SparkSubmitArguments类对象
- 执行submit()方法

```scala?linenums&fancy=5,11
 def doSubmit(args: Array[String]): Unit = {
    // Initialize logging if it hasn't been done yet. Keep track of whether logging needs to
    // be reset before the application starts.
    val uninitLog = initializeLogIfNecessary(true, silent = true)
    //>>==代码批注: 根据spark-submit提交的参数构造SparkSubmitArguments对象==<<
    val appArgs = parseArguments(args)
    if (appArgs.verbose) {
      logInfo(appArgs.toString)
    }
    appArgs.action match {
      >>==case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog)==<<
      case SparkSubmitAction.KILL => kill(appArgs)
      case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
      case SparkSubmitAction.PRINT_VERSION => printVersion()
    }
  }
```
接下来我们进入submit()
- 首先判断集群是否为standalone模式,这里由于集群是k8s native模式,直接执行else,进入doRunMain()
- 由于我们spark-submit没有`--proxy-user`,直接执行53行的else,进入runMain() 

``` scala?linenums&fancy=53
   /**
   * Submit the application using the provided parameters, ensuring to first wrap
   * in a doAs when --proxy-user is specified.
   */
  @tailrec
  private def submit(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {

    def doRunMain(): Unit = {
      if (args.proxyUser != null) {
        val proxyUser = UserGroupInformation.createProxyUser(args.proxyUser,
          UserGroupInformation.getCurrentUser())
        try {
          proxyUser.doAs(new PrivilegedExceptionAction[Unit]() {
            override def run(): Unit = {
              runMain(args, uninitLog)
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
        runMain(args, uninitLog)
      }
    }

    // In standalone cluster mode, there are two submission gateways:
    //   (1) The traditional RPC gateway using o.a.s.deploy.Client as a wrapper
    //   (2) The new REST-based gateway introduced in Spark 1.3
    // The latter is the default behavior as of Spark 1.3, but Spark submit will fail over
    // to use the legacy gateway if the master endpoint turns out to be not a REST server.
    if (args.isStandaloneCluster && args.useRest) {
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

```

进入runMain有两个关键步骤:
- 1. 初始化childArgs,childClasspath,sparkConf,childMainClass.
- 2. 实例化childMainClass

以上所谓的child都指的是resource manager中不用模式下提交作业的Client Main Class.

``` scala?linenums&fancy=14,55,57
 /**
   * Run the main method of the child class using the submit arguments.
   *
   * This runs in two steps. First, we prepare the launch environment by setting up
   * the appropriate classpath, system properties, and application arguments for
   * running the child main class based on the cluster manager and the deploy mode.
   * Second, we use this launch environment to invoke the main method of the child
   * main class.
   *
   * Note that this main class will not be the one provided by the user if we're
   * running cluster deploy mode or python applications.
   */
  private def runMain(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {
    val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)
    // Let the main class re-initialize the logging system once it starts.
    if (uninitLog) {
      Logging.uninitialize()
    }

    if (args.verbose) {
      logInfo(s"Main class:\n$childMainClass")
      logInfo(s"Arguments:\n${childArgs.mkString("\n")}")
      // sysProps may contain sensitive information, so redact before printing
      logInfo(s"Spark config:\n${Utils.redact(sparkConf.getAll.toMap).mkString("\n")}")
      logInfo(s"Classpath elements:\n${childClasspath.mkString("\n")}")
      logInfo("\n")
    }
    val loader = getSubmitClassLoader(sparkConf)
    for (jar <- childClasspath) {
      addJarToClasspath(jar, loader)
    }

    var mainClass: Class[_] = null

    try {
      mainClass = Utils.classForName(childMainClass)
    } catch {
      case e: ClassNotFoundException =>
        logError(s"Failed to load class $childMainClass.")
        if (childMainClass.contains("thriftserver")) {
          logInfo(s"Failed to load main class $childMainClass.")
          logInfo("You need to build Spark with -Phive and -Phive-thriftserver.")
        }
        throw new SparkUserAppException(CLASS_NOT_FOUND_EXIT_STATUS)
      case e: NoClassDefFoundError =>
        logError(s"Failed to load $childMainClass: ${e.getMessage()}")
        if (e.getMessage.contains("org/apache/hadoop/hive")) {
          logInfo(s"Failed to load hive class.")
          logInfo("You need to build Spark with -Phive and -Phive-thriftserver.")
        }
        throw new SparkUserAppException(CLASS_NOT_FOUND_EXIT_STATUS)
    }

    val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
      mainClass.getConstructor().newInstance().asInstanceOf[SparkApplication]
    } else {
      new JavaMainApplication(mainClass)
    }

```
下面开始详细说明runMain()的两步:
#### 第一步,初始化spark应用配置

``` scala
val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)
```
我们可以看一下prepareSubmitEnvironment()方法中有以下关键部分:

1. 确认集群模式
``` scala?linenums
 // Set the cluster manager
    val clusterManager: Int = args.master match {
      case "yarn" => YARN
      case m if m.startsWith("spark") => STANDALONE
      case m if m.startsWith("mesos") => MESOS
      case m if m.startsWith("k8s") => KUBERNETES
      case m if m.startsWith("local") => LOCAL
      case _ =>
        error("Master must either be yarn or start with spark, mesos, k8s, or local")
        -1
    }

    // Set the deploy mode; default is client mode
    var deployMode: Int = args.deployMode match {
      case "client" | null => CLIENT
      case "cluster" => CLUSTER
      case _ =>
        error("Deploy mode must be either client or cluster")
        -1
    }
```
确认完spark-submit提交的参数中是kubernetes的cluster模式之后
2. 封装spark应用的classpath,files,sparkConf,以及childmainClass.
prepareSubmitEnvironment()方法中关于k8s的几个代码块:
> 初始化k8s模式的spark master
```scala?linenums

    if (clusterManager == KUBERNETES) {
      args.master = Utils.checkAndGetK8sMasterUrl(args.master)
      // Make sure KUBERNETES is included in our build if we're trying to use it
      if (!Utils.classIsLoadable(KUBERNETES_CLUSTER_SUBMIT_CLASS) && !Utils.isTesting) {
        error(
          "Could not load KUBERNETES classes. " +
            "This copy of Spark may not have been compiled with KUBERNETES support.")
      }
    }
```
> 构造各种集群模式判断条件的`flag`变量

``` scala?linenums
    val isYarnCluster = clusterManager == YARN && deployMode == CLUSTER
    val isMesosCluster = clusterManager == MESOS && deployMode == CLUSTER
    val isStandAloneCluster = clusterManager == STANDALONE && deployMode == CLUSTER
    val isKubernetesCluster = clusterManager == KUBERNETES && deployMode == CLUSTER
    val isKubernetesClient = clusterManager == KUBERNETES && deployMode == CLIENT
    val isKubernetesClusterModeDriver = isKubernetesClient &&
      sparkConf.getBoolean("spark.kubernetes.submitInDriver", false)

```
当然,我们的集群模式是kubernetes的cluster模式,根据isKubernetesCluster和isKubernetesClusterModeDriver,进入特定jar依赖解决和下载远程文件的流程,如果是Yarn或者Messos则进入的是其他流程:
> 将依赖的Jar 加入classpath
```scala?linenums
if (!isMesosCluster && !isStandAloneCluster) {
      // Resolve maven dependencies if there are any and add classpath to jars. Add them to py-files
      // too for packages that include Python code
      val resolvedMavenCoordinates = DependencyUtils.resolveMavenDependencies(
        args.packagesExclusions, args.packages, args.repositories, args.ivyRepoPath,
        args.ivySettingsPath)

      if (!StringUtils.isBlank(resolvedMavenCoordinates)) {
        // In K8s client mode, when in the driver, add resolved jars early as we might need
        // them at the submit time for artifact downloading.
        // For example we might use the dependencies for downloading
        // files from a Hadoop Compatible fs eg. S3. In this case the user might pass:
        // --packages com.amazonaws:aws-java-sdk:1.7.4:org.apache.hadoop:hadoop-aws:2.7.6
        if (isKubernetesClusterModeDriver) {
          val loader = getSubmitClassLoader(sparkConf)
          for (jar <- resolvedMavenCoordinates.split(",")) {
            addJarToClasspath(jar, loader)
          }
        } else if (isKubernetesCluster) {
          // We need this in K8s cluster mode so that we can upload local deps
          // via the k8s application, like in cluster mode driver
          childClasspath ++= resolvedMavenCoordinates.split(",")
        } else {
          args.jars = mergeFileLists(args.jars, resolvedMavenCoordinates)
          if (args.isPython || isInternal(args.primaryResource)) {
            args.pyFiles = mergeFileLists(args.pyFiles, resolvedMavenCoordinates)
          }
        }
      }


```
> 下载依赖的远程文件
```scilab?linenums
    // In client mode, download remote files.
    var localPrimaryResource: String = null
    var localJars: String = null
    var localPyFiles: String = null
    if (deployMode == CLIENT) {
      localPrimaryResource = Option(args.primaryResource).map {
        downloadFile(_, targetDir, sparkConf, hadoopConf, secMgr)
      }.orNull
      localJars = Option(args.jars).map {
        downloadFileList(_, targetDir, sparkConf, hadoopConf, secMgr)
      }.orNull
      localPyFiles = Option(args.pyFiles).map {
        downloadFileList(_, targetDir, sparkConf, hadoopConf, secMgr)
      }.orNull

      if (isKubernetesClusterModeDriver) {
        // Replace with the downloaded local jar path to avoid propagating hadoop compatible uris.
        // Executors will get the jars from the Spark file server.
        // Explicitly download the related files here
        args.jars = renameResourcesToLocalFS(args.jars, localJars)
        val localFiles = Option(args.files).map {
          downloadFileList(_, targetDir, sparkConf, hadoopConf, secMgr)
        }.orNull
        args.files = renameResourcesToLocalFS(args.files, localFiles)
      }
    }

> 初始化sparkConf

``` scala?linenums
 // A list of rules to map each argument to system properties or command-line options in
    // each deploy mode; we iterate through these below
    val options = List[OptionAssigner](

      // All cluster managers
      OptionAssigner(args.master, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES, confKey = "spark.master"),
      OptionAssigner(args.deployMode, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
        confKey = SUBMIT_DEPLOY_MODE.key),
      OptionAssigner(args.name, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES, confKey = "spark.app.name"),
      OptionAssigner(args.ivyRepoPath, ALL_CLUSTER_MGRS, CLIENT, confKey = "spark.jars.ivy"),
      OptionAssigner(args.driverMemory, ALL_CLUSTER_MGRS, CLIENT,
        confKey = DRIVER_MEMORY.key),
      OptionAssigner(args.driverExtraClassPath, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
        confKey = DRIVER_CLASS_PATH.key),
      OptionAssigner(args.driverExtraJavaOptions, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
        confKey = DRIVER_JAVA_OPTIONS.key),
      OptionAssigner(args.driverExtraLibraryPath, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
        confKey = DRIVER_LIBRARY_PATH.key),
      OptionAssigner(args.principal, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
        confKey = PRINCIPAL.key),
      OptionAssigner(args.keytab, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
        confKey = KEYTAB.key),
      OptionAssigner(args.pyFiles, ALL_CLUSTER_MGRS, CLUSTER, confKey = SUBMIT_PYTHON_FILES.key),

      // Propagate attributes for dependency resolution at the driver side
      OptionAssigner(args.packages, STANDALONE | MESOS | KUBERNETES,
        CLUSTER, confKey = "spark.jars.packages"),
      OptionAssigner(args.repositories, STANDALONE | MESOS | KUBERNETES,
        CLUSTER, confKey = "spark.jars.repositories"),
      OptionAssigner(args.ivyRepoPath, STANDALONE | MESOS | KUBERNETES,
        CLUSTER, confKey = "spark.jars.ivy"),
      OptionAssigner(args.packagesExclusions, STANDALONE | MESOS | KUBERNETES,
        CLUSTER, confKey = "spark.jars.excludes"),

      // Yarn only
      OptionAssigner(args.queue, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.queue"),
      OptionAssigner(args.pyFiles, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.dist.pyFiles",
        mergeFn = Some(mergeFileLists(_, _))),
      OptionAssigner(args.jars, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.dist.jars",
        mergeFn = Some(mergeFileLists(_, _))),
      OptionAssigner(args.files, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.dist.files",
        mergeFn = Some(mergeFileLists(_, _))),
      OptionAssigner(args.archives, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.dist.archives",
        mergeFn = Some(mergeFileLists(_, _))),

      // Other options
      OptionAssigner(args.numExecutors, YARN | KUBERNETES, ALL_DEPLOY_MODES,
        confKey = EXECUTOR_INSTANCES.key),
      OptionAssigner(args.executorCores, STANDALONE | YARN | KUBERNETES, ALL_DEPLOY_MODES,
        confKey = EXECUTOR_CORES.key),
      OptionAssigner(args.executorMemory, STANDALONE | MESOS | YARN | KUBERNETES, ALL_DEPLOY_MODES,
        confKey = EXECUTOR_MEMORY.key),
      OptionAssigner(args.totalExecutorCores, STANDALONE | MESOS | KUBERNETES, ALL_DEPLOY_MODES,
        confKey = CORES_MAX.key),
      OptionAssigner(args.files, LOCAL | STANDALONE | MESOS | KUBERNETES, ALL_DEPLOY_MODES,
        confKey = FILES.key),
      OptionAssigner(args.jars, LOCAL, CLIENT, confKey = JARS.key),
      OptionAssigner(args.jars, STANDALONE | MESOS | KUBERNETES, ALL_DEPLOY_MODES,
        confKey = JARS.key),
      OptionAssigner(args.driverMemory, STANDALONE | MESOS | YARN | KUBERNETES, CLUSTER,
        confKey = DRIVER_MEMORY.key),
      OptionAssigner(args.driverCores, STANDALONE | MESOS | YARN | KUBERNETES, CLUSTER,
        confKey = DRIVER_CORES.key),
      OptionAssigner(args.supervise.toString, STANDALONE | MESOS, CLUSTER,
        confKey = DRIVER_SUPERVISE.key),
      OptionAssigner(args.ivyRepoPath, STANDALONE, CLUSTER, confKey = "spark.jars.ivy"),

      // An internal option used only for spark-shell to add user jars to repl's classloader,
      // previously it uses "spark.jars" or "spark.yarn.dist.jars" which now may be pointed to
      // remote jars, so adding a new option to only specify local jars for spark-shell internally.
      OptionAssigner(localJars, ALL_CLUSTER_MGRS, CLIENT, confKey = "spark.repl.local.jars")	  
    )
	//--------------------
	//忽略部分代码
	//--------------------
	 // Map all arguments to command-line options or system properties for our chosen mode
    for (opt <- options) {
      if (opt.value != null &&
          (deployMode & opt.deployMode) != 0 &&
          (clusterManager & opt.clusterManager) != 0) {
        if (opt.clOption != null) { childArgs += (opt.clOption, opt.value) }
        if (opt.confKey != null) {
          if (opt.mergeFn.isDefined && sparkConf.contains(opt.confKey)) {
            sparkConf.set(opt.confKey, opt.mergeFn.get.apply(sparkConf.get(opt.confKey), opt.value))
          } else {
            sparkConf.set(opt.confKey, opt.value)
          }
        }
      }
    }
```

> 初始化childMainClass,并传递Spark Application的mainClass或者R,Python的执行文件到childArgs

如果是Client模式,childMainClass直接就是Spark Application Main Class:

``` scala?linenums
// In client mode, launch the application main class directly
    // In addition, add the main application jar and any added jars (if any) to the classpath
    if (deployMode == CLIENT) {
      childMainClass = args.mainClass
      if (localPrimaryResource != null && isUserJar(localPrimaryResource)) {
        childClasspath += localPrimaryResource
      }
      if (localJars != null) { childClasspath ++= localJars.split(",") }
    }
```
如果是Cluster模式,childMainClass就是Kubernetes的Client Main Class,由它去调用Spark Application Main Class.
``` scala?linenums&fancy=2,3
    if (isKubernetesCluster) {
	  >>==//这里KUBERNETES_CLUSTER_SUBMIT_CLASS是指:==<<
      >>==//org.apache.spark.deploy.k8s.submit.KubernetesClientApplication,通过这个类来真正调用Spark应用的mainClass==<<	  
      childMainClass = KUBERNETES_CLUSTER_SUBMIT_CLASS
      if (args.primaryResource != SparkLauncher.NO_RESOURCE) {
        if (args.isPython) {
          childArgs ++= Array("--primary-py-file", args.primaryResource)
          childArgs ++= Array("--main-class", "org.apache.spark.deploy.PythonRunner")
        } else if (args.isR) {
          childArgs ++= Array("--primary-r-file", args.primaryResource)
          childArgs ++= Array("--main-class", "org.apache.spark.deploy.RRunner")
        }
        else {
          childArgs ++= Array("--primary-java-resource", args.primaryResource)
          childArgs ++= Array("--main-class", args.mainClass)
        }
      } else {
        childArgs ++= Array("--main-class", args.mainClass)
      }
      if (args.childArgs != null) {
        args.childArgs.foreach { arg =>
          childArgs += ("--arg", arg)
        }
      }
    }

```
最后完成childArgs, childClasspath, sparkConf, childMainClass的初始化并返回


#### 第二步,执行spark应用
进入runMain完成第一步之后,执行childClass中的main方法,这里cluster模式的childClass就是Yarn,Kubernetes,Mesos提交作业的Client类
通过Client再一次调用我们编写的Spark mainClass,这里使用例子SparkPi:

```scala
package org.apache.spark.examples

import scala.math.random

import org.apache.spark.sql.SparkSession

/** Computes an approximation to pi */
object SparkPi {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession
      .builder
      .appName("Spark Pi")
      .getOrCreate()
    val slices = if (args.length > 0) args(0).toInt else 2
    val n = math.min(100000L * slices, Int.MaxValue).toInt // avoid overflow
    val count = spark.sparkContext.parallelize(1 until n, slices).map { i =>
      val x = random * 2 - 1
      val y = random * 2 - 1
      if (x*x + y*y <= 1) 1 else 0
    }.reduce(_ + _)
    println(s"Pi is roughly ${4.0 * count / (n - 1)}")
    spark.stop()
  }
}
```
所以从spark-class执行java命令之后,调用关系链为SparkSubmit.main->KubernetesClientApplication.main->SparkPi.main
在知道调用关系链之后,我们再看runMain的最后片段,可以看到第20行调用了SparkApplication接口的start(childArgs,sparkConf)方法.
``` scala?linenums&fancy=20
  val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
      >>==//源码批注: 这里Yarn或者Kubernetes cluster模式会进入这里,因为Yarn/KubernetesClientApplication继承自SparkApplication==<<  
      mainClass.getConstructor().newInstance().asInstanceOf[SparkApplication]
    } else {
	  >>==//源码批注: Client模式会进入这里,直接invoke SparkApplication mainClass的main方法==<<
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
      app.start(childArgs.toArray, sparkConf)
    } catch {
      case t: Throwable =>
        throw findCause(t)
    }
```
现在我们再进入KubernetesClientApplication.start()方法:

``` scala?linenums
/**
 * Main class and entry point of application submission in KUBERNETES mode.
 */
private[spark] class KubernetesClientApplication extends SparkApplication {

  override def start(args: Array[String], conf: SparkConf): Unit = {
    val parsedArguments = ClientArguments.fromCommandLineArgs(args)
    run(parsedArguments, conf)
  }

  private def run(clientArguments: ClientArguments, sparkConf: SparkConf): Unit = {
    val appName = sparkConf.getOption("spark.app.name").getOrElse("spark")
    // For constructing the app ID, we can't use the Spark application name, as the app ID is going
    // to be added as a label to group resources belonging to the same application. Label values are
    // considerably restrictive, e.g. must be no longer than 63 characters in length. So we generate
    // a unique app ID (captured by spark.app.id) in the format below.
    val kubernetesAppId = s"spark-${UUID.randomUUID().toString.replaceAll("-", "")}"
    val waitForAppCompletion = sparkConf.get(WAIT_FOR_APP_COMPLETION)
    val kubernetesConf = KubernetesConf.createDriverConf(
      sparkConf,
      kubernetesAppId,
      clientArguments.mainAppResource,
      clientArguments.mainClass,
      clientArguments.driverArgs)
    // The master URL has been checked for validity already in SparkSubmit.
    // We just need to get rid of the "k8s://" prefix here.
    val master = KubernetesUtils.parseMasterUrl(sparkConf.get("spark.master"))
    val loggingInterval = if (waitForAppCompletion) Some(sparkConf.get(REPORT_INTERVAL)) else None

    val watcher = new LoggingPodStatusWatcherImpl(kubernetesAppId, loggingInterval)

    Utils.tryWithResource(SparkKubernetesClientFactory.createKubernetesClient(
      master,
      Some(kubernetesConf.namespace),
      KUBERNETES_AUTH_SUBMISSION_CONF_PREFIX,
      SparkKubernetesClientFactory.ClientType.Submission,
      sparkConf,
      None,
      None)) { kubernetesClient =>
        val client = new Client(
          kubernetesConf,
          new KubernetesDriverBuilder(),
          kubernetesClient,
          waitForAppCompletion,
          watcher)
        client.run()
    }
  }
}

```
这里由Client连接k8s的API Server,开始构建Kubernetes Driver Pod,提交Spark on k8s的作业.
接下来,我们开始分析Driver Pod又是如何构建Exector Pod,并分配作业的.
未完待续...
- [ ]  spark kubernetes 代码UML结构分析
- [ ] spark on k8s 作业调度流程
