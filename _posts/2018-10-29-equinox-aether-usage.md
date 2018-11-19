---
layout: post
title:  "eclipse aether用法"
categories: OSGi
tags:  aether maven
author: WuSanDi
---

* content
{:toc}





# 一、获取Aether
如果你想要用Aether做测试驱动，最容易的方式就是建立一个Maven工程。在你的POM中添加下面的XML就可以将Aether添加到你的类路径中：
```xml
<project>
  ...
  <properties>
    <aetherVersion>1.0.0.v20140518</aetherVersion>
    <mavenVersion>3.1.0</mavenVersion>
    <wagonVersion>1.0</wagonVersion>
  </properties>
  ...
  <dependencies>
    <dependency>
      <groupId>org.eclipse.aether</groupId>
      <artifactId>aether-api</artifactId>
      <version>${aetherVersion}</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.aether</groupId>
      <artifactId>aether-util</artifactId>
      <version>${aetherVersion}</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.aether</groupId>
      <artifactId>aether-impl</artifactId>
      <version>${aetherVersion}</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.aether</groupId>
      <artifactId>aether-connector-basic</artifactId>
      <version>${aetherVersion}</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.aether</groupId>
      <artifactId>aether-transport-file</artifactId>
      <version>${aetherVersion}</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.aether</groupId>
      <artifactId>aether-transport-http</artifactId>
      <version>${aetherVersion}</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.aether</groupId>
      <artifactId>aether-transport-wagon</artifactId>
      <version>${aetherVersion}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.maven</groupId>
      <artifactId>maven-aether-provider</artifactId>
      <version>${mavenVersion}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.maven.wagon</groupId>
      <artifactId>wagon-ssh</artifactId>
      <version>${wagonVersion}</version>
    </dependency>
  </dependencies>
  ...
</project>
```
让我们近距离看看这些依赖是干什么的：
  + aether-api
这个JAR包含Aether客户端使用的应用编程接口。系统的入口类是 org.eclipse.aether.RepositorySystem.
  + aether-util
这个JAR里是各种工具类和仓库系统收集的现成的组件。
  + aether-impl
这个JAR持有仓库系统的许多实际实现类。除非你想要自定义系统的内部实现或者需要手动将它们接在一起，客户端代码不应该直接访问这个JAR中的任何类。
  + aether-connector-basic
从远程仓库下载构件和上传构件到远处仓库都是由所谓的仓库连接器实现的。这个特殊的连接器是通用的，将一部分工作委派给可插拔的传输协议和仓库布局。所以一定要清楚，这个连接器自己是不能访问任何仓库的，所以用户总是需要包含一个或多个传输模块来获得功能系统。
  + aether-transport-file
传输模块，支持访问以"file:"开头的URL仓库
  + aether-transport-http
传输模块，支持访问"http:"和“https:”开头的仓库
  + aether-transport-wagon
传输模块，基于Maven Wagon ，可以雇佣任何已有的Wagon provider访问仓库。
  + wagon-ssh
这个JAR补充了之前提到的 aether-transport-wagon ，支持使用“scp:”和"sftp:"开头的仓库。上面的POM片段只是一个示例，Its inclusion in the above POM snippet is merely an example, use whatever Wagon providers fit your needs or none at all, in which case one can also drop aether-transport-wagon from the list.
  + maven-aether-provider
这个JAR提供了将Maven POM作为构件描述符，并从构件描述符中抽取依赖信息。此外，它还提供了处理其它Maven仓库用到的元数据文件。 
注意: Aether运行在Java 1.5+以上版本。所以请确保相应的编译设置。现在，你的工程拥有了所有Aether依赖，下面你可能想要知道如何设置Aether。

# 二、设置Aether
Aether的实现由一堆组件组成，这些组件需要绑在一起组成一个完整的仓库系统。为了满足不同的需求以及平滑集成到已经存在的应用中，Aether提供了多种方式来达到这个目的。本节假设你已经知道如何设置所需库的类路径，如果不知道，请首先阅读上一节。

JSR-330 / Guice / Sisu
依赖注入是组件装配的已经确定的方法，成为JSR-330的标准。Aether的组件自带了 javax.inject 注解，以支持在JSR-330应用中的用法。

Google Guice 是一个流行的JSR-330实现。为了在Guice应用中轻松的使用Aether，Aether提供了现成的Guice模块，它绑定了所有的常用实现类。参见runnable demo snippets了解使用Guice完整的例子。

对应那些想要让这些在类路径的组件被自动发现，而不是手动将它们绑定到一起的用户，Sisu可以帮上忙。此外，还有用自动或动态的组件发现扩展Guice。Aether的 JAR包自带了Sisu index文件，以优化组件发现的性能。此外的，我们的演示代码也提供了一个例子。

## 服务定位器
有时候，不希望使用完全成熟的依赖注入容器，它有一些开销。为了不用IoC容器获取仓库系统实例，Aether的默认实现类都提供了无参构造函数，提供了set方法手动将组件绑定在一起。但是由于这样做写代码很不爽，于是仓库系统又额外支持通过轻量的服务定位器模式绑定。
服务定位器基础设施只有两个接口，即：
```
org.eclipse.aether.spi.locator.Service 
org.eclipse.aether.spi.locator.ServiceLocator
```
组件自己实现 Service 接口，仓库系统的客户端提供ServiceLocator的实现，以便从这里查询其它的组件。
Apache Maven工程的模块 maven-aether-provider 为Aether提供了一个简单的服务定位器，它预先填充了需要处理Maven仓库的组件。注意，Eclipse Aether需要 maven-aether-provider 3.1.0及以上版 (早期版本依赖于Sonatype Aether)。上述的服务定位器的用法看起来像下面这样：
```java
import org.apache.maven.repository.internal.MavenRepositorySystemUtils;
import org.eclipse.aether.connector.wagon.WagonProvider;
import org.eclipse.aether.connector.wagon.WagonRepositoryConnectorFactory;
...
    private static RepositorySystem newRepositorySystem()  {
        DefaultServiceLocator locator = MavenRepositorySystemUtils.newServiceLocator();
        locator.addService( RepositoryConnectorFactory.class, BasicRepositoryConnectorFactory.class );
        locator.addService( TransporterFactory.class, FileTransporterFactory.class );
        locator.addService( TransporterFactory.class, HttpTransporterFactory.class );
        return locator.getService( RepositorySystem.class );
    }
...
```
通过MavenRepositorySystemUtils创建的DefaultServiceLocator已经知道来自 aether-impl 和 maven-aether-provider的所有组件，所以其它需要告知的就只剩下0个或多个仓库连接器和传输器。一旦服务定位器内部的服务注册表计算完成，那么调用 getService() 方法就会创建仓库系统，并递归初始化它的子组件。

# 三、创建仓库系统会话
Aether和它的组件都被设计成无状态的，所有的这些配置或状态都被传递给了方法。当多个请求同时解析依赖时，相当多的设置对于这些方法调用通常都是相同的，，例如代理设置或本地仓库的路径。 那些对于整个仓库系统会话都相同的设置由  org.eclipse.aether.RepositorySystemSession 实例表示。使用maven-aether-provider,中的类，创建这样的会话模仿Maven的设置的代码如下所示：
```java
import org.apache.maven.repository.internal.MavenRepositorySystemUtils;
...
    private static RepositorySystemSession newSession( RepositorySystem system ) {
        DefaultRepositorySystemSession session = MavenRepositorySystemUtils.newSession();
        LocalRepository localRepo = new LocalRepository( "target/local-repo" );
        session.setLocalRepositoryManager( system.newLocalRepositoryManager( session, localRepo ) );
        return session;
    }
```
正如你看到的，唯一必须指定的设置是本地仓库，其它的设置都是初始化为默认值。请查看DefaultRepositorySystemSession API文档学些其它的会话配置。
如果你更近一步的探索与Apache Maven的合作，想要读取用户目录的settings.xml文件，你应该看看 org.apache.maven:maven-settings-builder 库，它提供了必要的信息。来自Aether Ant任务的 AntRepoSys.getSettings() 方法可以为你自己的代码提供一些灵感。但是请直接在Maven的邮件列表中提问有关该库的用法的问题。

# 四、解析依赖
继续扩展第二节和第三节的代码片段，下面的代码演示了如何实际解析传递依赖，例如 org.apache.maven:maven-profile:2.2.1 ，并将结果作为类路径输出到控制器：
```java
    public static void main( String[] args ) throws Exception {
        RepositorySystem repoSystem = newRepositorySystem();
        RepositorySystemSession session = newSession( repoSystem );
        Dependency dependency =  
new Dependency( new DefaultArtifact( "org.apache.maven:maven-profile:2.2.1" ), "compile" );
        RemoteRepository central = 
new RemoteRepository.Builder( "central", "default", "http://repo1.maven.org/maven2/" ).build();
        CollectRequest collectRequest = new CollectRequest();
        collectRequest.setRoot( dependency );
        collectRequest.addRepository( central );
        DependencyNode node = repoSystem.collectDependencies( session, collectRequest ).getRoot();
        DependencyRequest dependencyRequest = new DependencyRequest();
        dependencyRequest.setRoot( node );
        repoSystem.resolveDependencies( session, dependencyRequest  );
        PreorderNodeListGenerator nlg = new PreorderNodeListGenerator();
        node.accept( nlg );
        System.out.println( nlg.getClassPath() );
    }
```
所以，一旦你初始化了仓库系统，并创建了一个仓库系统会话，通用模式就是创建一些请求对象，调用它的set方法配置请求，执行操作并鉴定结果对象。
由于“所有的理论都是灰色的”，我们在代码中维护了一些可运行的例子。这些例子提供了一个更多Aether的扩展说明，所以你在等什么？
要学习如何控制器传递依赖的计算过程，你可以看看《Aether入门》中“传递依赖解析”一节。

# 五、在Maven插件中使用Aether
Apache Maven 3.x使用Aether执行仓库任务，且那些面向Maven3.x的插件也可以这样做。开始前，你可能想要在你的POM中添加下面的依赖：
```xml
<project>
  ...
  <prerequisites>
    <!-- Maven 3.1.0 is the earliest version using Eclipse Aether, Maven 3.0.x uses the incompatible predecessor Sonatype Aether -->
    <maven>3.1</maven>
  </prerequisites>
 
  <dependencies>
    <dependency>
      <!-- required in all cases -->
      <groupId>org.eclipse.aether</groupId>
      <artifactId>aether-api</artifactId>
      <version>0.9.0.M2</version>
    </dependency>
    <dependency>
      <!-- optional helpers, might be superfluous depending on your use case -->
      <groupId>org.eclipse.aether</groupId>
      <artifactId>aether-util</artifactId>
      <version>0.9.0.M2</version>
    </dependency>
    <!--
    WARNING: Beware of http://jira.codehaus.org/browse/MNG-5513 which is triggered by a direct or transitive dependency on aether-impl in compile or runtime scope.
    -->
    ...
  </dependencies>
  ...
</project>
```
注意：在运行时， aether-api的实际版本是Maven核心强制约束的，与其它Maven API相同。所以确保编译/测试你的插件时，针对 aether-api 的版本要求满足插件支持的Maven的最低版本要求。
接下来，在你的Mojo源码中，你需要抓取仓库有关的组件和参数：
```java
import org.eclipse.aether.RepositorySystem;
import org.eclipse.aether.RepositorySystemSession;
import org.eclipse.aether.repository.RemoteRepository;
...
public MyMojo extends AbstractMojo { 
    /**
     * The entry point to Aether, i.e. the component doing all the work.
     * 
     * @component
     */
    private RepositorySystem repoSystem;
    /**
     * The current repository/network configuration of Maven.
     * 
     * @parameter default-value="${repositorySystemSession}"
     * @readonly
     */
    private RepositorySystemSession repoSession;
    /**
     * The project's remote repositories to use for the resolution of project dependencies.
     * 
     * @parameter default-value="${project.remoteProjectRepositories}"
     * @readonly
     */
    private List<RemoteRepository> projectRepos;
    /**
     * The project's remote repositories to use for the resolution of plugins and their dependencies.
     * 
     * @parameter default-value="${project.remotePluginRepositories}"
     * @readonly
     */
    private List<RemoteRepository> pluginRepos;
    // Your other mojo parameters and code here
    ...
}
```
通常，你只需要 projectRepos 或 pluginRepos ，它们依赖于你的插件要处理的构件的种类，所以其它的插件参数都是多余的。但是通常，在Maven插件内部，上面的代码应该给出使用Aether的处理逻辑。


