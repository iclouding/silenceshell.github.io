---
layout: post
title: "yarn（二）：distributedShell和Unmanaged AM示例代码解析"
date: 2015-05-05 16:56:22
tags: yarn hadoop
author: 伊布
categories: tech
cover:  "/assets/instacode.png"
---


hadoop源码中，使用yarn的应用程序除了MR以外，还有2个示例程序，我们先来分析distributedShell，顺带介绍YARN应用程序设计方法。
以下代码基于2.6.0版本。

###一 示例执行
我使用ambari安装的hadoop环境，jar包在`/usr/lib/hadoop-yarn`中。
执行命令：

```bash
$ su hdfs
$ hadoop jar hadoop-yarn-applications-distributedshell-2.2.0.2.0.6.0-101.jar org.apache.hadoop.yarn.applications.distributedshell.Client -jar hadoop-yarn-applications-distributedshell-2.2.0.2.0.6.0-101.jar  -shell_command '/bin/date' -num_containers 10
```

需要切到hdfs用户，否则会有下面的错误提示：

```bash
15/05/05 09:17:42 INFO distributedshell.Client: Copy App Master jar from local filesystem and add to local environment
15/05/05 09:17:43 FATAL distributedshell.Client: Error running CLient
org.apache.hadoop.security.AccessControlException: Permission denied: user=root, access=WRITE, inode="/user":hdfs:hdfs:drwxr-xr-x
```

原因是本地hdfs上的`/user`目录只对hdfs用户开放了写权限，root不可写。cloudera安装的时候可以选择*所有服务使用同一个账户*，不会存在权限的问题（但据说会造成安装变复杂）。
执行完后提示信息请见文章最后。

###二 Client解析
distShell主要有2个类组成，Client和ApplicationMaster。两个类都带有main入口。Client的主要工作是启动AM，真正要做的任务由AM来调度。
Client的简化框架如下。

```java
  public static void main(String[] args) {
    boolean result = false;
    try {
      Client client = new Client();  //1 创建Client对象
      try {
        boolean doRun = client.init(args);  //2 初始化
        if (!doRun) {
          System.exit(0);
        }
      }
      result = client.run();   //3 运行
    }
    if (result) {
      System.exit(0);
    }
    System.exit(2);
  }
```

####1 创建Client对象
创建时会指定本Client要用到的AM。
创建yarnClient。yarn将client与RM的交互抽象出了编程库YarnClient，用以应用程序提交、状态查询和控制等，简化应用程序。

```java
  public Client(Configuration conf) throws Exception  {
    this(		//指定AM
      "org.apache.hadoop.yarn.applications.distributedshell.ApplicationMaster",
      conf);
  Client(String appMasterMainClass, Configuration conf) {
    this.conf = conf;
    this.appMasterMainClass = appMasterMainClass;
    yarnClient = YarnClient.createYarnClient();		//创建yarnClient
    yarnClient.init(conf);
    opts = new Options();	//创建opts，后面解析参数的时候用
    opts.addOption("appname", true, "Application Name. Default value - DistributedShell");
    opts.addOption("priority", true, "Application Priority. Default 0");
}
```


####2 初始化
init会解析命令行传入的参数，例如使用的jar包、内存大小、cpu个数等。
代码里使用GnuParser解析：init时定义所有的参数opts（可以认为是一个模板），然后将opts和实际的args传入解析后得到一个CommnadLine对象，后面查询选项直接操作该CommnadLine对象即可，如`cliParser.hasOption("help")`和`cliParser.getOptionValue("jar")`。

```java
  public boolean init(String[] args) throws ParseException {
    CommandLine cliParser = new GnuParser().parse(opts, args);
    amMemory = Integer.parseInt(cliParser.getOptionValue("master_memory", "10"));
    amVCores = Integer.parseInt(cliParser.getOptionValue("master_vcores", "1"));
    shellCommand = cliParser.getOptionValue("shell_command");
    appMasterJar = cliParser.getOptionValue("jar");
    ...
```

#### 3 运行
- 先启动yarnClient，会建立跟RM的RPC连接，之后就跟调用本地方法一样。通过此yarnClient查询NM个数、NM详细信息（ID/地址/Container个数等）、Queue info（其实没用到，示例里只是打印了下调试用）。

```java
public class Client {
  public boolean run() throws IOException, YarnException {
    yarnClient.start();
    YarnClusterMetrics clusterMetrics = yarnClient.getYarnClusterMetrics();
    List<NodeReport> clusterNodeReports = yarnClient.getNodeReports(
```

- 收集提交AM所需的信息。

```java
    YarnClientApplication app = yarnClient.createApplication();	//创建app
    GetNewApplicationResponse appResponse = app.getNewApplicationResponse();
...
    ApplicationSubmissionContext appContext = app.getApplicationSubmissionContext();
    //AM需要的本地资源，如jar包、log文件
    Map<String, LocalResource> localResources = new HashMap<String, LocalResource>();

    FileSystem fs = FileSystem.get(conf);
    addToLocalResources(fs, appMasterJar, appMasterJarPath, appId.toString(),
        localResources, null);
    ...	//添加localResource

    vargs.add(Environment.JAVA_HOME.$$() + "/bin/java");
    vargs.add("-Xmx" + amMemory + "m");
    vargs.add(appMasterMainClass);
...
    for (CharSequence str : vargs) {
      command.append(str).append(" ");	//重新组织命令行
    }
	//创建Container加载上下文，包含本地资源，环境变量，实际命令。
    ContainerLaunchContext amContainer = ContainerLaunchContext.newInstance(
      localResources, env, commands, null, null, null);

    Resource capability = Resource.newInstance(amMemory, amVCores);
    appContext.setResource(capability);		//请求使用的内存、cpu

    appContext.setAMContainerSpec(amContainer);
    appContext.setQueue(amQueue);
```

重新组织出来的commands如下：

```bash
$JAVA_HOME/bin/java -Xmx10m org.apache.hadoop.yarn.applications.distributedshell.ApplicationMaster --container_memory 10
```
- 提交AM（即appContext），并启动监控。
Client只关心自己提交到RM的AM是否正常运行，而AM内部的多个task，由AM管理。如果Client要查询应用程序的任务信息，需要自己设计与AM的交互。
```java
    yarnClient.submitApplication(appContext);   //客户端提交AM到RM
    return monitorApplication(appId);
```

总的来说，Client做的事情比较简单，即建立与RM的连接，提交AM，监控AM运行状态。
> 有个疑问，走读代码没有看到jar包是怎么送到NM上去的。

###三 Application Master解析

AM简化框架如下：

```java
      boolean doRun = appMaster.init(args);
      if (!doRun) {
        System.exit(0);
      }
      appMaster.run();
      result = appMaster.finish();
```

yarn抽象了两个编程库，AMRMClient和NMClient(AM和RM都可以用)，简化AM编程。

####1 设置RM、NM消息的异步处理方法

```java
    AMRMClientAsync.CallbackHandler allocListener = new RMCallbackHandler();
    amRMClient = AMRMClientAsync.createAMRMClientAsync(1000, allocListener);
    amRMClient.init(conf);
    amRMClient.start();

    containerListener = createNMCallbackHandler();
    nmClientAsync = new NMClientAsyncImpl(containerListener);
    nmClientAsync.init(conf);
    nmClientAsync.start();
```

####2 向RM注册

```java
    RegisterApplicationMasterResponse response = amRMClient.registerApplicationMaster(appMasterHostname,
        appMasterRpcPort, appMasterTrackingUrl);
```

####3 计算需要的Container，向RM发起请求

```java
    // Setup ask for containers from RM
    // Send request for containers to RM
    // Until we get our fully allocated quota, we keep on polling RM for
    // containers
    // Keep looping until all the containers are launched and shell script
    // executed on them ( regardless of success/failure).
    for (int i = 0; i < numTotalContainersToRequest; ++i) {
      ContainerRequest containerAsk = setupContainerAskForRM();
      amRMClient.addContainerRequest(containerAsk);		//请求指定个数的Container
    }

  private ContainerRequest setupContainerAskForRM() {
    Resource capability = Resource.newInstance(containerMemory,
      containerVirtualCores);		//指定需要的memory/cpu能力
    ContainerRequest request = new ContainerRequest(capability, null, null,
        pri);

```

好吧，先假设上面的`addContainerRequest`会向RM发送请求。对于AM来说，接下来就是等待RM回消息告知分配的Container。
> Q：注释里说这里会一直循环，怎么理解？按说发起Container请求以后，异步等待RM的应答，在相应的处理中加载任务（前面已经注册了AMRM的回调方法）就行了。

####4 RM分配Container给AM，AM启动任务
**RMCallbackHandler**
RM消息的响应，由`RMCallbackHandler`处理。示例中主要对前两种消息进行了处理。

```java
  private class RMCallbackHandler implements AMRMClientAsync.CallbackHandler {
    //处理消息：Container执行完毕。在RM返回的心跳应答中携带。如果心跳应答中有已完成和新分配两种Container，先处理已完成
    public void onContainersCompleted(List<ContainerStatus> completedContainers) {
...
    //处理消息：RM新分配Container。在RM返回的心跳应答中携带
    public void onContainersAllocated(List<Container> allocatedContainers) {

    public void onShutdownRequest() {done = true;}

    //节点状态变化
    public void onNodesUpdated(List<NodeReport> updatedNodes) {}

    public float getProgress() {
```

`onContainersAllocated`收到分配的Container之后，会提交任务到NM。

```java
    public void onContainersAllocated(List<Container> allocatedContainers) {
        LaunchContainerRunnable runnableLaunchContainer =   //创建runnable容器
            new LaunchContainerRunnable(allocatedContainer, containerListener);
        Thread launchThread = new Thread(runnableLaunchContainer);	//新建线程

        // launch and start the container on a separate thread to keep
        // the main thread unblocked
        // as all containers may not be allocated at one go.
        launchThreads.add(launchThread);
        launchThread.start();	//线程中提交Container到NM，不影响主流程
```

简单分析下`LaunchContainerRunnable`。该类实现自Runnable，其run方法准备任务命令（本例即为`date`）。

```java
  private class LaunchContainerRunnable implements Runnable {
    public LaunchContainerRunnable(
        Container lcontainer, NMCallbackHandler containerListener) {
      this.container = lcontainer;		//创建时记录待使用的Container
      this.containerListener = containerListener;
    }
    public void run() {
      vargs.add(shellCommand);		//待执行的shell命令
      vargs.add(shellArgs);			//shell命令参数
      List<String> commands = new ArrayList<String>();
      commands.add(command.toString());	//转为commands

      //根据命令、环境变量、本地资源等创建Container加载上下文
      ContainerLaunchContext ctx = ContainerLaunchContext.newInstance(
              localResources, shellEnv, commands, null, allTokens.duplicate(), null);
      containerListener.addContainer(container.getId(), container);
      //异步启动Container
      nmClientAsync.startContainerAsync(container, ctx);
```

`onContainersCompleted`的功能比较简单，收到Container执行完毕的消息，检查其执行结果，如果执行失败，则重新发起请求，直到全部完成。

**NMCallbackHandler**
NM消息的响应，由`NMCallbackHandler`处理。

在distShell示例里，回调句柄对NM通知过来的各种事件的处理比较简单，只是修改AM维护的Container执行完成、失败的个数。这样等到有Container执行完毕后，可以重启发起请求。失败处理和上面Container执行完毕消息的处理类似，达到了上面问题里所说的loopback效果。

```java
  static class NMCallbackHandler
    implements NMClientAsync.CallbackHandler {

    @Override
    public void onContainerStopped(ContainerId containerId) {

    @Override
    public void onContainerStatusReceived(ContainerId containerId,

    @Override
    public void onContainerStarted(ContainerId containerId,
...
```

总的来说，AM做的事就是向RM/NM注册回调函数，然后请求Container；得到Container后提交任务，并跟踪这些任务的执行情况，如果失败了则重新提交，直到全部任务完成。

###四 UnmanagedAM
distShell的Client提交AM到RM后，由RM将AM分配到某一个NM上的Container，这样给AM调试带来了困难。yarn提供了一个参数，Client可以设置为Unmanaged，提交AM后，会在客户端本地起一个单独的进程来运行AM。

```java
public class UnmanagedAMLauncher {
  public void launchAM(ApplicationAttemptId attemptId)
    //创建新进程
    Process amProc = Runtime.getRuntime().exec(amCmd, envAMList.toArray(envAM));
    try {
      int exitCode = amProc.waitFor();  //等待AM进程结束
    } finally {
      amCompleted = true;
    }

  public boolean run() throws IOException, YarnException {
      appContext.setUnmanagedAM(true);		//设置为Unmanaged
      rmClient.submitApplication(appContext);	//提交AM

      ApplicationReport appReport =		//监控AM状态，如果状态变为ACCEPTED，则跳出循环，launchAM。
          monitorApplication(appId, EnumSet.of(YarnApplicationState.ACCEPTED,
            YarnApplicationState.KILLED, YarnApplicationState.FAILED,
            YarnApplicationState.FINISHED));

      if (appReport.getYarnApplicationState() == YarnApplicationState.ACCEPTED) {
        launchAM(attemptId);
```


附：命令执行输出

```bash
15/05/05 09:12:22 INFO distributedshell.Client: Initializing Client
15/05/05 09:12:22 INFO distributedshell.Client: Running Client
15/05/05 09:12:23 INFO client.RMProxy: Connecting to ResourceManager at ty11.dtdream.com/10.168.250.59:8050
15/05/05 09:12:23 INFO distributedshell.Client: Got Cluster metric info from ASM, numNodeManagers=3
15/05/05 09:12:23 INFO distributedshell.Client: Got Cluster node info from ASM
15/05/05 09:12:23 INFO distributedshell.Client: Got node report from ASM for, nodeId=ty11.dtdream.com:45454, nodeAddressty11.dtdream.com:8042, nodeRackName/default-rack, nodeNumContainers0
15/05/05 09:12:23 INFO distributedshell.Client: Got node report from ASM for, nodeId=ty10.dtdream.com:45454, nodeAddressty10.dtdream.com:8042, nodeRackName/default-rack, nodeNumContainers0
15/05/05 09:12:23 INFO distributedshell.Client: Got node report from ASM for, nodeId=ty12.dtdream.com:45454, nodeAddressty12.dtdream.com:8042, nodeRackName/default-rack, nodeNumContainers0
15/05/05 09:12:23 INFO distributedshell.Client: Queue info, queueName=default, queueCurrentCapacity=0.0, queueMaxCapacity=1.0, queueApplicationCount=0, queueChildQueueCount=0
15/05/05 09:12:23 INFO distributedshell.Client: User ACL Info for Queue, queueName=root, userAcl=SUBMIT_APPLICATIONS
15/05/05 09:12:23 INFO distributedshell.Client: User ACL Info for Queue, queueName=root, userAcl=ADMINISTER_QUEUE
15/05/05 09:12:23 INFO distributedshell.Client: User ACL Info for Queue, queueName=default, userAcl=SUBMIT_APPLICATIONS
15/05/05 09:12:23 INFO distributedshell.Client: User ACL Info for Queue, queueName=default, userAcl=ADMINISTER_QUEUE
15/05/05 09:12:23 INFO distributedshell.Client: Max mem capabililty of resources in this cluster 4096
15/05/05 09:12:23 INFO distributedshell.Client: Copy App Master jar from local filesystem and add to local environment
15/05/05 09:12:23 INFO distributedshell.Client: Set the environment for the application master
15/05/05 09:12:23 INFO distributedshell.Client: Setting up app master command
15/05/05 09:12:23 INFO distributedshell.Client: Completed setting up app master command $JAVA_HOME/bin/java -Xmx10m org.apache.hadoop.yarn.applications.distributedshell.ApplicationMaster --container_memory 10 --num_containers 10 --priority 0 --shell_command /bin/date  1><LOG_DIR>/AppMaster.stdout 2><LOG_DIR>/AppMaster.stderr
15/05/05 09:12:23 INFO distributedshell.Client: Submitting application to ASM
15/05/05 09:12:23 INFO impl.YarnClientImpl: Submitted application application_1430207548681_0011 to ResourceManager at ty11.dtdream.com/10.168.250.59:8050
15/05/05 09:12:24 INFO distributedshell.Client: Got application report from ASM for, appId=11, clientToAMToken=null, appDiagnostics=, appMasterHost=N/A, appQueue=default, appMasterRpcPort=-1, appStartTime=1430788343925, yarnAppState=ACCEPTED, distributedFinalState=UNDEFINED, appTrackingUrl=http://ty11.dtdream.com:8088/proxy/application_1430207548681_0011/, appUser=hdfs
15/05/05 09:12:25 INFO distributedshell.Client: Got application report from ASM for, appId=11, clientToAMToken=null, appDiagnostics=, appMasterHost=ty10.dtdream.com/10.252.142.223, appQueue=default, appMasterRpcPort=-1, appStartTime=1430788343925, yarnAppState=RUNNING, distributedFinalState=UNDEFINED, appTrackingUrl=http://ty11.dtdream.com:8088/proxy/application_1430207548681_0011/, appUser=hdfs
...
15/05/05 09:12:32 INFO distributedshell.Client: Got application report from ASM for, appId=11, clientToAMToken=null, appDiagnostics=, appMasterHost=ty10.dtdream.com/10.252.142.223, appQueue=default, appMasterRpcPort=-1, appStartTime=1430788343925, yarnAppState=FINISHED, distributedFinalState=SUCCEEDED, appTrackingUrl=http://ty11.dtdream.com:8088/proxy/application_1430207548681_0011/, appUser=hdfs
15/05/05 09:12:32 INFO distributedshell.Client: Application has completed successfully. Breaking monitoring loop
15/05/05 09:12:32 INFO distributedshell.Client: Application completed successfully

```

