---
layout: post
title:  "pax-url mvn协议 maven指导"
categories: OSGi
tags:  OPS4J URL
author: WuSanDi
---

* content
{:toc}





起初，这篇文档发布在Using Maven with OSGi, part 1

pax-url-aether 库让在OSGi运行时内执行Maven任务更加容易。最重要的任务之一就是构件解析，大体上就是一个将maven坐标变成物理资源的过程。这个资源可能会安装到OSGi运行时（例如OSGi的bundle）。

我个人认为学习任何技术的内部原理是长期使用和维护它最好的方式。最后下载官方源代码，在IDE中阅读源代码，比官方或者非官方的文档都要好。

当然，有时候（通常）没有时间去深挖内部机制，所以我希望这篇文章可以提供一个选择。

# Maven
虽然有更好的选择(如 Gradle)， Apache Maven 依然是事实上的构建和依赖管理的标准工具。当我们将MAVEN分解成不同的部分时，我们可以在我们的代码中重用依赖管理。依赖管理是软件开发和OSGi运行时一个很重要的方面。Maven依赖只是依赖管理层次之一。OSGi bundle和Karaf feature也可以看作其它种类的依赖。

我们想要在运行时拉取存储在外部Maven仓库上的任意构件并使用它(例如bundle, feature 或配置文件) 。我们想要以Maven方式完成它，即声明方式。最佳的例子是在OSGi运行时安装外部的Maven构件。例如Karaf:
```shell
karaf@root()> bundle:install mvn:commons-io/commons-io/2.5
Bundle ID: 52
```
这些命令开箱即用，但是通常需要改变一些默认配置，例如配置额外的远程仓库、修改证书，配置HTTP代理等等。配置选项有专门的页面介绍。这里我们从基本的Aether library库开始介绍。
# Aether
Eclipse Aether 是一组Maven内部解析依赖的库。使用Aether可以执行各种任务，如依赖图中找到构件的闭包。但是即使使用这些低级的库，我们也要关注一个特殊的任务：从远程仓库获取构件。
Official Aether Wiki page对于入门已经足够了，可以看看如何在代码使用它。我会提供更多详细的信息来描述一些重要的概念。
Aether使用基于接口的API，而这些API的实际实现都是用CDI（或者依赖注入）配置的。有两个重要的接口：
  + org.eclipse.aether.RepositorySystem - 仓库系统的入口，提供各种不同的解析方法
  + org.eclipse.aether.RepositorySystemSession - 提供额外的特殊信息给RepositorySystem上执行的操作
有一组类：
  + org.eclipse.aether.*.*Request - 传入的各种请求类，作为命令传递给RepositorySystem.<br>
  我们将主要关注 on org.eclipse.aether.resolution.ArtifactRequest.
RepositorySystem 以依赖注入的风格配置的，我们可以选择几个SPI接口的具体实现，它们会改变Aether的一些方面。而 RepositorySystemSession 是用属性和直接set方法配置的。Session 改变的是仓库处理请求的方式。
所以让我们看看这些是如何一起工作的。首先让我们配置repository system:
```java
DefaultServiceLocator locator = MavenRepositorySystemUtils.newServiceLocator();
locator.setService(RepositoryConnectorFactory.class, BasicRepositoryConnectorFactory.class);
locator.setService(TransporterFactory.class, FileTransporterFactory.class);
locator.setService(TransporterFactory.class, HttpTransporterFactory.class);
locator.setService(org.eclipse.aether.spi.log.LoggerFactory.class, Slf4jLoggerFactory.class);
RepositorySystem system = locator.getService(RepositorySystem.class);
```
没什么了不起的：我们与权限访问 http:// 和 file:// 开头的仓库，使用SLF4J API记录日志。
现在，让我们配置 session。配置属性是任意的，更多属性在稍后介绍。
```java
RepositorySystemSession session = MavenRepositorySystemUtils.newSession();
((DefaultRepositorySystemSession)session).setConfigProperty("aether.connector.basic.threads", "2");
LocalRepositoryManager localRepositoryManager =
          system.newLocalRepositoryManager(session, new LocalRepository("/home/me/.m2/repository"));
((DefaultRepositorySystemSession)session).setLocalRepositoryManager(localRepositoryManager);
```
最后，让我们执行一些操作，构件解析：
```java
ArtifactRequest req = new ArtifactRequest();
req.setArtifact(new DefaultArtifact("commons-io", "commons-io", "jar", "2.5"));
req.addRepository(new RemoteRepository.Builder("central", "default", "http://repo1.maven.org/maven2").build());
ArtifactResult res = system.resolveArtifact(session, req);
```
上面的代码告诉Aether用在本地仓库 /home/user/.m2/repository 中解析构件 commons-io:commons-io:jar:2.5 ，如果没有找到，就在远程仓库  http://repo1.maven.org/maven2 搜索该构件。我们可以配置更多的远程仓库（这是通常的做法）： org.eclipse.aether.resolution.ArtifactRequest#addRepository()) 。
上面的代码不需要使用Karaf内的Maven或JBoss Fuse，但是它带来了两个超级重要的概念：
  + 本地仓库 - Aether通过 org.eclipse.aether.repository.LocalRepositoryManager 接口和 org.eclipse.aether.repository.LocalRepository 类访问。实际上本地仓库是可访问的本地文件系统目录的包装，它遵循特定的结构(Maven构件的组织方式)。
  + 远程仓库 - Aether通过 org.eclipse.aether.repository.RemoteRepository 接口访问。实际上远程仓库是URI、一组与snapshot/release版本号有关的策略加上代理、镜像和鉴权信息的包装。
关键是如果在本地仓库找不到构件，它会在远程仓库中搜索该构件。正规代码都应该确保本地仓库总是在远程仓库前被搜索。

# 日志
为了调试，在日志中看到所有的操作是很有帮助的。我们可以增加一些日志记录器的级别（在Karaf中的配置文件为etc/org.ops4j.pax.logging.cfg):
```
log4j.logger.org.eclipse.aether = DEBUG
log4j.logger.org.apache.http.headers = DEBUG
log4j.logger.shaded.org.eclipse.aether = DEBUG
log4j.logger.shaded.org.apache.http.headers = DEBUG
```
这些带有shaded的日志记录器是由必要的，因为在一些第三方库中使用的pax-url-aether的私有版本，如Aether 或httpclient。
另外，我们添加另外一个仓库看看Aether是怎么检查它们的：
```java
req.addRepository(new RemoteRepository.Builder("jboss-public", "default", "https://repository.jboss.org/nexus/content/groups/public").build());
req.addRepository(new RemoteRepository.Builder("central", "default", "http://repo1.maven.org/maven2").build());
```
下面是解析 commons-io:commons-io:2.5:jar 构件时的日志，它在本地仓库中不可用：
```
11:13:47.181
 DEBUG {main} [o.e.a.i.i.DefaultLocalRepositoryProvider] : Using manager
 EnhancedLocalRepositoryManager with priority 10.0 for 
target/repo-1469178827169
11:13:47.188 INFO  {main} [g.t.m.a.AetherTest] : Request: commons-io:commons-io:jar:2.5 < [jboss-public (https://repository.jboss.org/nexus/content/groups/public, default, releases+snapshots), central (http://repo1.maven.org/maven2, default, releases+snapshots)]
11:13:47.631 DEBUG {main} [o.e.a.i.i.DefaultTransporterProvider] : Using transporter HttpTransporter with priority 5.0 for https://repository.jboss.org/nexus/content/groups/public
11:13:47.632
 DEBUG {main} [o.e.a.i.i.DefaultRepositoryConnectorProvider] : Using 
connector BasicRepositoryConnector with priority 0.0 for https://repository.jboss.org/nexus/content/groups/public
11:13:49.015
 DEBUG {main} [o.a.h.headers] : >> GET 
/nexus/content/groups/public/commons-io/commons-io/2.5/commons-io-2.5.jar
 HTTP/1.1
11:13:49.015 DEBUG {main} [o.a.h.headers] : >> Host: repository.jboss.org
...
11:13:49.385 DEBUG {main} [o.a.h.headers] : << HTTP/1.1 404 Not Found
...
11:13:49.572 DEBUG {main} [o.e.a.i.i.DefaultTransporterProvider] : Using transporter HttpTransporter with priority 5.0 for http://repo1.maven.org/maven2
11:13:49.572
 DEBUG {main} [o.e.a.i.i.DefaultRepositoryConnectorProvider] : Using 
connector BasicRepositoryConnector with priority 0.0 for http://repo1.maven.org/maven2
11:13:49.704 DEBUG {main} [o.a.h.headers] : >> GET /maven2/commons-io/commons-io/2.5/commons-io-2.5.jar HTTP/1.1
11:13:49.705 DEBUG {main} [o.a.h.headers] : >> Host: repo1.maven.org
...
11:13:49.770 DEBUG {main} [o.a.h.headers] : << HTTP/1.1 200 OK
...
11:13:50.079 DEBUG {main} [o.a.h.headers] : >> GET /maven2/commons-io/commons-io/2.5/commons-io-2.5.jar.sha1 HTTP/1.1
11:13:50.079 DEBUG {main} [o.a.h.headers] : >> Host: repo1.maven.org
...
11:13:50.145 DEBUG {main} [o.a.h.headers] : << HTTP/1.1 200 OK
...
11:13:50.156
 DEBUG {main} [o.e.a.i.i.EnhancedLocalRepositoryManager] : Writing 
tracking file 
/data/ggrzybek/sources/_testing/grgr-test-maven/target/repo-1469178827169/commons-io/commons-io/2.5/_remote.repositories
11:13:50.161 INFO  {main} [g.t.m.a.AetherTest] : Result: commons-io:commons-io:jar:2.5 < central (http://repo1.maven.org/maven2, default, releases+snapshots)
```
正如我们看到的，这里发生了一系列事件：
  1. Aether使用了位于 target/repo-1469178827169 的本地仓库
  2. 首先检查的是https://repository.jboss.org/nexus/content/groups/public ，我们得到却是404错误
  3. 下一个检查的是http://repo1.maven.org/maven2，我们得到是HTTP 200
  4. 然后Aether拉取找到的构件的SHA1检验文件
  5. Aether写入跟踪文件target/repo-1469178827169/commons-io/commons-io/2.5/_remote.repositories ，内容长这个样子：
#NOTE: This is an Aether internal implementation file, its format can be changed without prior notice.
#Fri Jul 22 11:13:50 CEST 2016
commons-io-2.5.jar>central=
这个文件允许我们重新访问下载构件的地方。

快照版本（SNAPSHOT）
让我们看看在解析SNAPSHOT版本时，Aether是如何工作的。我们将重用前面的远程仓库。默认情况下 new RemoteRepository.Builder("central", "default", "http://repo1.maven.org/maven2").build() 构造的远程仓库，不管我们是否使用这个仓库来解析 SNAPSHOT和非SNAPSHOT构件，它都使能了。当然，我们可以修改它：
```java
RemoteRepository.Builder b1 = new RemoteRepository.Builder("central", "default", "http://repo1.maven.org/maven2");
RemoteRepository.Builder b2 = new RemoteRepository.Builder("jboss-public", "default", "https://repository.jboss.org/nexus/content/groups/public");
RepositoryPolicy enabledPolicy = new RepositoryPolicy(true, RepositoryPolicy.UPDATE_POLICY_ALWAYS, RepositoryPolicy.CHECKSUM_POLICY_FAIL);
RepositoryPolicy disabledPolicy = new RepositoryPolicy(false, RepositoryPolicy.UPDATE_POLICY_ALWAYS, RepositoryPolicy.CHECKSUM_POLICY_FAIL);
b1.setReleasePolicy(enabledPolicy);
b1.setSnapshotPolicy(enabledPolicy);
b2.setReleasePolicy(disabledPolicy);
b2.setSnapshotPolicy(enabledPolicy);
req.addRepository(b1.build());
req.addRepository(b2.build());
```
上面的示例中，我们明确地使能从central和jboss-public仓库解析SNAPSHOT构件。我们不会尝试从 jboss-public. 解析非SNAPSHOT构件。下面是解析 commons-io:commons-io:2.5-SNAPSHOT:jar日志：
```
12:11:17.195 DEBUG {main} [o.e.a.i.i.DefaultLocalRepositoryProvider] : Using manager EnhancedLocalRepositoryManager with priority 10.0 for target/repo-1469182277187
12:11:17.201 INFO  {main} [g.t.m.a.AetherTest] : Request: commons-io:commons-io:jar:2.5-SNAPSHOT < [central (http://repo1.maven.org/maven2, default, releases+snapshots), jboss-public (https://repository.jboss.org/nexus/content/groups/public, default, snapshots)]
12:11:17.851 DEBUG {DefaultMetadataResolver-0-1} [o.e.a.i.i.DefaultTransporterProvider] : Using transporter HttpTransporter with priority 5.0 for https://repository.jboss.org/nexus/content/groups/public
12:11:17.852 DEBUG {DefaultMetadataResolver-0-1} [o.e.a.i.i.DefaultRepositoryConnectorProvider] : Using connector BasicRepositoryConnector with priority 0.0 for https://repository.jboss.org/nexus/content/groups/public
12:11:17.853 DEBUG {DefaultMetadataResolver-0-0} [o.e.a.i.i.DefaultTransporterProvider] : Using transporter HttpTransporter with priority 5.0 for http://repo1.maven.org/maven2
12:11:17.854 DEBUG {DefaultMetadataResolver-0-0} [o.e.a.i.i.DefaultRepositoryConnectorProvider] : Using connector BasicRepositoryConnector with priority 0.0 for http://repo1.maven.org/maven2
12:11:18.158 DEBUG {DefaultMetadataResolver-0-0} [o.a.h.headers] : >> GET /maven2/commons-io/commons-io/2.5-SNAPSHOT/maven-metadata.xml HTTP/1.1
12:11:18.158 DEBUG {DefaultMetadataResolver-0-0} [o.a.h.headers] : >> Host: repo1.maven.org
...
12:11:18.225 DEBUG {DefaultMetadataResolver-0-0} [o.a.h.headers] : << HTTP/1.1 404 Not Found
...
12:11:18.245 DEBUG {DefaultMetadataResolver-0-0} [o.e.a.i.i.DefaultUpdateCheckManager] : Writing tracking file /data/ggrzybek/sources/_testing/grgr-test-maven/target/repo-1469182277187/commons-io/commons-io/2.5-SNAPSHOT/resolver-status.properties
12:11:19.332 DEBUG {DefaultMetadataResolver-0-1} [o.a.h.headers] : >> GET /nexus/content/groups/public/commons-io/commons-io/2.5-SNAPSHOT/maven-metadata.xml HTTP/1.1
12:11:19.332 DEBUG {DefaultMetadataResolver-0-1} [o.a.h.headers] : >> Host: repository.jboss.org
...
12:11:19.611 DEBUG {DefaultMetadataResolver-0-1} [o.a.h.headers] : << HTTP/1.1 200 OK
...
12:11:19.850 DEBUG {DefaultMetadataResolver-0-1} [o.a.h.headers] : >> GET /nexus/content/groups/public/commons-io/commons-io/2.5-SNAPSHOT/maven-metadata.xml.sha1 HTTP/1.1
12:11:19.850 DEBUG {DefaultMetadataResolver-0-1} [o.a.h.headers] : >> Host: repository.jboss.org
...
12:11:20.079 DEBUG {DefaultMetadataResolver-0-1} [o.a.h.headers] : << HTTP/1.1 200 OK
...
12:11:20.082 DEBUG {DefaultMetadataResolver-0-1} [o.e.a.i.i.DefaultUpdateCheckManager] : Writing tracking file /data/ggrzybek/sources/_testing/grgr-test-maven/target/repo-1469182277187/commons-io/commons-io/2.5-SNAPSHOT/resolver-status.properties
12:11:20.107 DEBUG {main} [o.e.a.i.i.DefaultTransporterProvider] : Using transporter HttpTransporter with priority 5.0 for https://repository.jboss.org/nexus/content/groups/public
12:11:20.107 DEBUG {main} [o.e.a.i.i.DefaultRepositoryConnectorProvider] : Using connector BasicRepositoryConnector with priority 0.0 for https://repository.jboss.org/nexus/content/groups/public
12:11:20.694 DEBUG {main} [o.a.h.headers] : >> GET /nexus/content/groups/public/commons-io/commons-io/2.5-SNAPSHOT/commons-io-2.5-20151119.212356-154.jar HTTP/1.1
12:11:20.694 DEBUG {main} [o.a.h.headers] : >> Host: repository.jboss.org
...
12:11:20.901 DEBUG {main} [o.a.h.headers] : << HTTP/1.1 200 OK
...
12:11:21.590 DEBUG {main} [o.e.a.i.i.EnhancedLocalRepositoryManager] : Writing tracking file /data/ggrzybek/sources/_testing/grgr-test-maven/target/repo-1469182277187/commons-io/commons-io/2.5-SNAPSHOT/_remote.repositories
12:11:21.591 INFO  {main} [g.t.m.a.AetherTest] : Result: commons-io:commons-io:jar:2.5-20151119.212356-154 < jboss-public (https://repository.jboss.org/nexus/content/groups/public, default, snapshots)
```
下面事件顺序：
  1. Aether使用本地仓库 target/repo-1469182277187 
  2. 并行从两个远程仓库拉取commons-io/commons-io/2.5-SNAPSHOT/maven-metadata.xml 元数据构件
  3. Metadata只在 jboss-public 仓库找到了
  4. Aether拉取metadata SHA1校验码
  5. 写入有关元数据的跟踪信息target/repo-1469182277187/commons-io/commons-io/2.5-SNAPSHOT/resolver-status.properties 
  6. target/repo-1469182277187/commons-io/commons-io/2.5-SNAPSHOT/maven-metadata-jboss-public.xml 文件显示 2.5-20151119.212356-154 是SNAPSHOT构件的最新版本
  7. Aether从jboss-public下载commons-io/commons-io/2.5-SNAPSHOT/commons-io-2.5-20151119.212356-154.jar 
  8. Aether写入跟踪文件 target/repo-1469182277187/commons-io/commons-io/2.5-SNAPSHOT/_remote.repositories ，它看起来长这样子：
```
#NOTE: This is an Aether internal implementation file, its format can be changed without prior notice.
#Fri Jul 22 12:11:21 CEST 2016
commons-io-2.5-20151119.212356-154.jar>jboss-public=
```
这个文件允许重新访问下载构件的仓库。

还有一些值得注意的事情：这次Aether在不同的线程中执行操作(DefaultMetadataResolver-0-* threads in addition to mainthread)。Aether通常在一次执行多个任务时这样做。<br>
在非SNAPSHOT构件解析中，我们一次只检查一个仓库，因为我们只有一个任务 - org.eclipse.aether.resolution.ArtifactRequest<br>
在SNAPSHOT构件解析中，Aether内部调用了两个 org.eclipse.aether.resolution.MetadataRequest 任务 (每一个仓库一个)来查找最新的SNAPSHOT。<br>
这个操作使用的线程数可以通过配置属性aether.metadataResolver.threads来控制。<br>
# 总结
本文提供了一些背景、低级信息和一些基本概念，例如本地仓库和远程仓库。 Mvn Protocol 提供了有关pax-url-aether自己的信息。
maven
aether
