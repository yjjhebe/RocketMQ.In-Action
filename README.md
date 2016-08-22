# rocketmq.in-action
what is rocketmq? 
why use rocketmq?
what is nameserver?
what is broker?
the introduction of rocketmq

# Name Server
Name Server 主要负责管理集群中所有的Topic队列信息和Broker地址信息，客户端可以通过NameServer获取topic信息，通过topic获取broker信息，通过broker获取broker地址信息等等。

## NamesrvStartup
启动Name Server，通过调用NamesrvStartup类的main方法进行创建，该类首先加载系统默认配置文件，NamesrvConfig和NettyServerConfig，顾名思义NamesrcConfig就是Name Server相关的配置信息，NettyServerConfig就是启动Netty时的相关配置信息。RocketMQ是使用Netty作为底层RPC通信框架的。
```Java
package com.alibaba.rocketmq.namesrv;

public class NamesrvStartup {

  /**
   * 程序或命令行启动，调用的入口
   * @param args
   */
  public static void main(String[] args) {
    main0(args);
  }

  public static NamesrcController main0(String[] args) {
    //...
    
    //创建NamesrvConfig配置文件
    final NamesrvConfig namesrvConfig = new NamesrvConfig();
    //创建NettyServerConfig配置文件
    final NettyServerConfig nettyServerConfig = new NettyServerConfig();
    nettyServerConfig.setListenPort(9876);//设置Netty服务端监听端口，默认9876
    
    //创建NamesrvController，传入配置文件构造controller
    final NamesrvController controller = new NamesrvController(namesrcConfig, nettyServerConfig);
    controller.initialize();//初始化controller
    
    //...
  }
  
}
```
NamesrvController是实际执行Name Server的地方，NamesrcStartup创建了NamesrvConfig和NettyServerConfig配置文件后，通过这两个配置文件实例化了NamesrvController控制器类，然后调用NamesrvController.initialize方法进行初始化。
## NamesrvController
NamesrvController实例化了RouteInfoManager和BrokerHouseKeepingService两个对象。Name Server中最重要的就是RouteInfoManager类。
```Java
package com.alibaba.rocketmq.namesrv;

public class NamesrvController {
  //Name Server 配置文件
  private final NamesrvCofnig namesrvConfig;
  //Netty 服务端配置文件
  private final NettyServerConfig nettyServerConfig;
  //Netty 服务端
  private RemotingServer remotingServer;
  //Broker状态监听处理程序
  private BrokerHouseKeepingServer brokerHouseKeepingService;
  //线程池 用于处理消息请求DefaultRequestProcessor
  private ExecutorService remotingService;
  
  //...
  
  /**
     * 实例化NamesrvController
     * @param namesrvConfig Name Server 配置文件
     * @param nettyServerConfig Netty 服务端配置文件
     */
  public NamesrvController(NamesrvConfig namesrvConfig, NettyServerConfig nettyServerConfig) {
    this.namesrvConfig = namesrvConfig;
    this.nettyServerConfig = nettyServerConfig;
    //...
    this.routeInfoManager = new RouteInfoManager();
    this.brokerHousekeepingService = new BrokerHousekeepingService(this);
  }
  
  /**
   * 初始化
   * @return
   */
  public boolean initialize() {
    //...
    
    //创建Netty服务端
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
    //创建一个固定大小的线程池，根据nettyServerConfig配置文件提供的默认工作线程数，默认值为8
    this.remotingExecutor = Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new      ThreadFactoryImpl("RemotingExecutorThread_"));
    //注册消息处理程序
    this.registerProcessor();
    
    //...
  }
}
```
### RouteInfoManager
RouteInfoManager就是管理topic和broker真正的地方
```Java
  package com.alibaba.rocketmq.namesrv.routeinfo;

  public class RouteInfoManager {
    //Topic列表信息
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    //Broker地址信息
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    //Broker集群信息
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    //Broker更新信息
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    //Broker过滤信息
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
    
    //...
  }
```
### BrokerHouseKeepingService
BrokerHouseKeepingService专门处理broker是否存活，如果broker失效或异常，则将broker从RouteInfoManager移除。同时将与该broker相关的topic信息也一起删除。
```Java
  package com.alibaba.rocketmq.namesrv.routeinfo;

  public class BrokerHouseKeepingService implements ChannelEventListener {
  
    //...
    
    @Override
    public void onChannelClose(String remoteAddr, Channel channel) {
      //管道关闭时，将broker从RouteInfoManager中移除
      this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }
    
    @Override
    public void onChannelException(String remoteAddr, Channel channel) {
      //管道异常时，将broker从RouteInfoManager中移除
      this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }

    @Override
    public void onChannelIdle(String remoteAddr, Channel channel) {
      //管道失效时，将broker从RouteInfoManager中移除
      this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }
    
    //...
  }
```
```Java
  package com.alibaba.rocketmq.namesrv.routeinfo;

  public class RouteInfoManager {

    //...
    
    /**
     * 销毁管道
     * @param remoteAddr 客户端地址信息
     * @param channel 客户端管道
     */
    public void onChannelDestroy(String remoteAddr, Channel channel) {
    
      //1.根据channel客户端管道找到brokerAddr对应的broker地址信息
      //2.根据brokerAddr地址将broker从brokerLiveTable和filterServerTable移除
      this.brokerLiveTable.remove(brokerAddrFound);
      this.filterServerTable.remove(brokerAddrFound);
      //3.根据brokerAddr遍历brokerAddrTable，将broker从brokerAddrTable中移除
      this.brokerAddrTable.remove();
      //4.根据brokerAddr遍历clusterAddrTable，将broker从clusterAddrTable中移除
      this.clusterAddrTable.remove();
      //4.根据brokerAddr遍历topicQueueTable，将broker从topicQueueTable中移除
      this.topicQueueTable.remove();
      
    }
  }
```
### NettyRemotingServer
同时NamesrvController创建了Netty服务端NettyRemotingServer，根据NamesrvStartup对象提供的nettyServerConfig配置文件，以及将BrokerHouseKeepingService处理程序传入NettyRemotingServer的构造函数中
```Java
  //nettyServerConfig Netty服务端相关配置文件，例如前面在NamesrvStartup中配置了监听端口9876
  //brokerHouseKeepingServer broker失效处理程序，
  this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHouseKeepingServer);
```
### RemotingServer
NamesrvController还创建了一个remotingExecutor线程池，用于处理Netty服务端接收到消息请求
```Java
  //创建了固定大小的线程池，根据nettyServerConfig配置文件提供的默认工作线程数，默认值为8
  this.remotingExecutor = Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
```
### DefaultRequestProcessor
NamesrvController为NettyRemotingServer注册了消息请求处理器DefaultRequestProcessor，当Netty服务端接收到消息请求时，调用remotingExecutor线程池执行DefaultRequestProcessor处理程序，DefaultRequestProcessor根据消息类型来做出相应的处理
```Java
  //为netty注册了默认的处理程序，DefaultRequestProcessor，以及用于执行该处理程序的线程池remotingExecutor
  this.remotingServer.registerDefaultProcessor(new DefaultRequestProcessor(this), this.remotingExecutor);
```
## DefaultRequestProcessor
DefaultRequestProcessor实际处理消息请求的类，请求的消息类型有：TOPIC，BROKER等。
```Java
  switch (request.getCode()) {
    case RequestCode.REGISTER_BROKER:
    case RequestCode.UNREGISTER_BROKER:
    case RequestCode.GET_ROUTEINTO_BY_TOPIC:
    case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
  }
```
根据不同的消息类型，操作RouteInfoManager管理的相应HashMap。
