---
layout: post
title:  "pax-web启动jetty服务器"
categories: OSGi
tags:  OPS4J Web   
author: WuSanDi
---

* content
{:toc}





# 一、pax-web-jetty Activator启动start方法

（注册ServerControllerFactory）
  1. 实例化ServerControllerFactoryImpl，并注册为OSGi服务ServerControllerFactory
  2. ServiceTracker跟踪Handler、Connector、Customizer，并分别调用ServerControllerFactoryImpl的addHandler、addConnector、addCustomizer添加jetty的Server中

## ServerControllerImpl配置和启动过程
     （实例化时传入了JettyFactoryImpl实例）
      a. 调用其configure方法，并传入org.ops4j.pax.web.service.spi.Configuration实例执行配置过程
      b. 将传入的方法参数保存到configuration变量，并调用state状态(内部类Unconfigured)的configure方法
      c. 在Unconfigured的configure方法中，更新状态为内部类Stopped，通知监听器ServerEvent.CONFIGURED事件，并调用ServerControllerImpl的start方法
      d. 在ServerControllerImpl的start方法（被重写）中，先向JettyServer添加Connector，然后继续调用state状态(内部类Stopped)的start方法，最后向JettyServer添加处理Handler、Customizer
      e. 在Stopped的start方法中，调用jettyFactory的createServer方法创建org.ops4j.pax.web.service.jetty.internal.jettyServer对象（实现类为JettyServerImpl，其持有继承自org.eclipse.jetty.server.Server的JettyServerWrapper），并用configuration对象对jettyServer进行配置（包括配置NCSA RequestLogHandler），最后调用jettyServer.start()方法调用JettyServer
      f. 更新状态为内部类Started，并通知监听器ServerEvent.STARTED事件

## JettyServerImpl start启动过程分析
      a. 查找jetty.xml配置：URL jettyResource
      b. 实例化XmlConfiguration configuration = new XmlConfiguration(jettyResource) 
      c. 反射XmlConfiguration的configure方法，并invoke JettyServerWrapper对象
      d. 调用JettyServerWrapper对象(继承自org.eclipse.jetty.server.Server)的start方法，启动Jetty Server服务端

# 二、pax-web-runntime Activator启动start方法
  （真正启动Jetty Server的地方）
  1. 实例化org.ops4j.pax.web.service.internal.ServletEventDispatcher
  2. 检查org.osgi.service.event.EventAdmin.class是否可用，如果可用，则ServiceTracker跟踪EventAdmin，并注册OSGi服务注册ServletListener 实例org.ops4j.pax.web.service.internal.EventAdminHandler
  3. 检查org.osgi.service.log.LogService.class是否可用，如果可用，则ServiceTracker跟踪LogService，并注册OSGi服务ServletListener实例org.ops4j.pax.web.service.internal.LogServiceHandler
  4. 检查org.osgi.service.cm.ManagedService.class是否可用，如果可用，则注册OSGi服务ManagedService： scheduleUpdateConfig（final Dictionary<String, ?> configuration），它的服务配置PID为“org.ops4j.pax.web”，对应的配置文件为org.ops4j.pax.web.cfg:
     
     javax.servlet.context.tempdir = data/pax-web-jsp
     org.ops4j.pax.web.config.file = etc/jetty.xml
     org.osgi.service.http.port = 8181
     #org.osgi.service.http.port.secure = 8543
     #org.osgi.service.http.secure.enabled = true
     
  5. 如果ConfigurationAdmin服务可用，则4中的ManagedService实例会执行updateController(Dictionary<String, ?> dictionary, ServerControllerFactory controllerFactory)
  6. 如果initialConfigSet为false，则将配置dictionary保存到config字段，同时ServiceTracker跟踪ServerControllerFactory服务，其回调函数为DynamicsServiceTrackerCustomizer
  7. 如果ServerControllerFactory服务可用，执行DynamicsServiceTrackerCustomizer回调，在addingService方法中，执行scheduleUpdateFactory
  8. 在scheduleUpdateFactory中，异步执行updateController(Dictionary<String, ?> dictionary, ServerControllerFactory controllerFactory)
  9. 这时initialConfigSet为true，由于controllerFactory不为null，
      <pre>
      a. 实例化DefaultPropertyResolver
      b. 实例化BundleContextPropertyResolver
      c. 实例化org.ops4j.util.property.DictionaryPropertyResolver(位于ops4j-base-util-property中)
      d. 实例化org.ops4j.pax.web.service.internal.ConfigurationImpl configuration
      e. 实例化ServerModel serverModel
      f. 调用controllerFactory.createServerController(serverModel)方法创建serverController，实现类为org.ops4j.pax.web.service.jetty.internal.ServerControllerImpl
      g. 用configuration实例配置serverController
      h. determineServiceProperties
      i. 注册一个具有Bundle作用域的ServiceFactory OSGi服务实例HttpServiceFactoryImpl，它的createService方法会创建org.ops4j.pax.web.service.internal.HttpServiceProxy对象（它实现了org.ops4j.pax.web.service.WebContainer接口，而WebContainer又继承了HttpService接口，实际上持有org.ops4j.pax.web.service.internal.HttpServiceStarted委派对象）
      j. 判断serverController.isStarted()是否启动，如果没有启动，则继续判断serverController.isConfigured()是否已经配置好，如果配置好了， 则调用serverController.start()方法启动server controller
      k. createManagedServiceFactory，实例化HttpContextProcessing对象，并注册为OSGi服务ManagedServiceFactory，用于创建HttpContext，这主要用在注册额外的Web条目，如初始配置和过滤器等
      </pre>

# 三、pax-web-extender-war Activator启动start方法

  1. 实例化WebEventDispatcher webEventDispatcher
  2. ServiceTracker跟踪org.osgi.service.packageadmin.PackageAdmin
  3. 实例化DefaultWebAppDependencyManager
  4. 实例化WebObserver（实例化WebAppParser， 实例化WebAppPublisher）
  5. 启动BundleTracker跟踪Bundle.ACTIVE

  
## BundleTracker跟踪到Bundle的Web注册过程
  6. 一旦某个Bundle状态变为Active，那么addingBundle->modifiedBundle->createExtension方法就会执行
  7. 在createExtension方法中调用doCreateExtension方法创建org.ops4j.pax.web.extender.war.internal.extender.Extension，doCreateExtension交给WebObserver.createExtension(Bundle)创建org.ops4j.pax.web.extender.war.internal.extender.SimpleExtension匿名类实例
具体执行过程：
      <pre>
      a. 检查Bundle的"Web-ContextPath"元数据头，即是否为WEB Bundle
      b. 创建org.ops4j.pax.web.extender.war.internal.model.WebApp实例
      c. eventDispatcher发布WebEvent.DEPLOYING事件
      d. WebAppParser解析Bundle，从Bundle中抽取"Webapp-Root"和"WEB-INF/web.xml"信息，并设置到WebApp实例中
      e. 将WebApp实例添加到DefaultWebAppDependencyManager实例中
      f. 注册WebApp
      g. 返回SimpleExtension实例
      </pre>
  8. 以同步方式或者异步方式调用extension.start()方法
  9. 在SimpleExtension start方法中，调用doStart方法，接着执行WebObserver.deploy(WebApp)方法
  10. 在WebObserver的deploy方法中，WebAppPublisher.pulish(WebApp)
  11. 在WebAppPublisher的pulish方法中，ServiceTracker跟踪WebAppDependencyHolder
  12. 一旦跟踪到WebAppDependencyHolder，就会只执行回调WebAppDependencyListener.addingService方法，接着调用register方法中
  13. 在WebAppDependencyListener.register方法中，判断HttpService是否为 org.ops4j.pax.web.service.WebContainer实例，如果是 org.ops4j.pax.web.service.WebContainer实例，则实例化RegisterWebAppVisitorWC访问WebApp， 否则实例化RegisterWebAppVisitorHS访问WebApp
  14. RegisterWebAppVisitorWC的visit执行过程：
      <pre>
      a. RegisterWebAppVisitorWC visit WebApp
      b. 实例化ResourceDelegatingBundleClassLoader
      c. 实例化WebAppWebContainerContext， 即HttpContext
      d. WebApp设置HttpContext
      e. webContainer设置上下文参数
      f. webContainer设置登录配置
      g. webContainer设置会话超时时间
      h. webContainer注册WebAppServletContainerInitializer
      i. webContainer设置虚拟主机
      j. webContainer设置Connectors
      k. webContainer注册JettyWebXml配置URL
      l. webContainer启动
      m. webContainer注册资源
      n. webContainer注册welcomeFile
      o. webContainer注册JSP支持
      p. webContainer注册WebAppListener监听器
      q. webContainer注册WebAppFilter过滤器
      r. webContainer注册WebAppServlet
      s. webContaine注册WebAppConstraintMapping
      t. webContainer注册WebAppErrorPage
      </pre>

  15. WebApp设置发布状态为WebEvent.DEPLOYED，eventDispatcher发布WebEvent.DEPLOYED事件


# 四、pax-web-deployer Activator启动start方法
  1. 实例化WarDeployer：Apache Felix FileInstall用该类将war文件转换为URL
  2. 注册OSGi服务：ArtifactUrlTransformer，ArtifactListener


