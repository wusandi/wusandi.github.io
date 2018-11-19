---
layout: post
title:  "pax-url mvn协议 aether配置"
categories: OSGi
tags:  OPS4J Web   
author: WuSanDi
---

* content
{:toc}





# 默认仓库
org.ops4j.pax.url.mvn.defaultRepositories - 这个配置是一组由逗号分开的本地仓库的列表，解析构件的第一阶段就是从本地仓库搜索。这个仓库只包含file://开头的URI。这个列表中的每一个仓库都应该当作本地仓库对待。pax-url-aether迭代遍历整个列表，一次检查一个仓库。如果这些仓库都没有包含要解析的构件，那么pax-url-aether就会切换到第二阶段，通过远程仓库解析。这些仓库也需要写权限-Aether不会在这里写任何文件。

用下面的代码可以访问这些仓库(参见Maven指导的更多低级背景):
```java
RepositorySystem system = locator.getService(RepositorySystem.class);
RepositorySystemSession session = MavenRepositorySystemUtils.newSession();
String basedir = singleRepositoryFromListOfDefaultRepositories;
((DefaultRepositorySystemSession)session).setLocalRepositoryManager(
            system.newLocalRepositoryManager(session, new LocalRepository(basedir)));
ArtifactRequest req = new ArtifactRequest();
req.setArtifact(new DefaultArtifact("commons-io", "commons-io", "jar", "2.5"));
ArtifactResult res = system.resolveArtifact(session, req);
```
我们不会调用任何的req.addRepository(repositoryBuilder.build()), 所以Aether不会尝试访问任何外部的位置。
singleRepositoryFromListOfDefaultRepositories 意思是一次只尝试访问org.ops4j.pax.url.mvn.defaultRepositories中的一个仓库。每一个这样的本地仓库都是独立检查的。

# 本地仓库
org.ops4j.pax.url.mvn.localRepository - 这个配置是本地仓库，支持Aether在构件解析的第二阶段。它的角色跟 org.ops4j.pax.url.mvn.defaultRepositories 有点不同。当Aether实际在某个远程仓库解析构件时，它会存储下载下来的构件到 org.ops4j.pax.url.mvn.localRepository （隐式位置）指定的位置。这也是为什么整个位置需要写权限。

如果没有指定，那么这个属性的默认值是 ${user.home}/.m2/repository。 明白这个是有用的，如果Aether没有找到期望的构件。

构件解析的第二阶段可以用下面的代码呈现：
```java
RepositorySystem system = locator.getService(RepositorySystem.class);
RepositorySystemSession session = MavenRepositorySystemUtils.newSession();
String basedir = localRepository;
((DefaultRepositorySystemSession)session).setLocalRepositoryManager(
           system.newLocalRepositoryManager(session, new LocalRepository(basedir)));
ArtifactRequest req = new ArtifactRequest();
req.addRepository(new RemoteRepository.Builder("ID1", "default", "http://uri1").build());
req.addRepository(new RemoteRepository.Builder("ID2", "default", "http://uri2").build());
req.setArtifact(new DefaultArtifact("commons-io", "commons-io", "jar", "2.5"));
ArtifactResult res = system.resolveArtifact(session, req);
```
这里，对于org.ops4j.pax.url.mvn.repositories中的每一个远程仓库，我们会调用org.eclipse.aether.resolution.ArtifactRequest.addRepository()方法添加仓库位置。
另外，basedir = localRepository 的意思是我们使用来自属性 org.ops4j.pax.url.mvn.localRepository 指定的本地仓库，每一个被搜索的远程仓库都使用的是相同的本地仓库。

# 远程仓库
org.ops4j.pax.url.mvn.repositories - 这个配置是在构件解析的第二阶段搜索构件使用的远程仓库列表。这个仓库可能会包含file:// 开头的URI，但是最好把这样的仓库添加到org.ops4j.pax.url.mvn.defaultRepositories属性中。 访问每一个仓库都需要使用配置的连接器(pax-url-aether使用的连接器的底层是调用httpclient 4.x库 - 这也是为什么要配置  org.apache.http.headers/shaded.org.apache.http.headers logger是一个好主意)。
如果 org.ops4j.pax.url.mvn.repositories 属性中的远程仓库列表是由+号作前缀，那么在settings.xml文件中定义的active profile中所有可用的仓库都会添加到远程仓库的有效列表。
例如，如果我们的配置如下：

```shell
org.ops4j.pax.url.mvn.repositories= \
    +http://repo1.maven.org/maven2@id=maven.central.repo
```

settings.xml文件如下:

```xml
<!--    If org.ops4j.pax.url.mvn.repositories property is _prepended_ with '+' sign, repositories from all active
    profiles will be _appended_ to the list of searched remote repositories-->
<profiles>
    <profile>
        <id>default</id>
        <repositories>
            <repository>
                <id>private.repository</id>
                <url>http://localhost:8181/maven-repository</url>
            </repository>
        </repositories>
    </profile>
</profiles>
<activeProfiles>
    <activeProfile>default</activeProfile>
</activeProfiles>
```

我们在解析过程中可以看到如下的日志：

```
18:10:17,734 | DEBUG | ... | Using transporter WagonTransporter with priority -1.0 for http://repo1.maven.org/maven2/
18:10:17,736 | DEBUG | ... | Using connector BasicRepositoryConnector with priority 0.0 for http://repo1.maven.org/maven2/
18:10:17,800 | DEBUG | ... | http-outgoing-8 >> GET /maven2/commons-io/commons-io/2.7/commons-io-2.7.jar HTTP/1.1
18:10:17,802 | DEBUG | ... | http-outgoing-8 >> Host: repo1.maven.org
...
18:10:17,872 | DEBUG | ... | Using transporter WagonTransporter with priority -1.0 for http://localhost:8181/maven-repository/
18:10:17,873 | DEBUG | ... | Using connector BasicRepositoryConnector with priority 0.0 for http://localhost:8181/maven-repository/
18:10:17,875 | DEBUG | ... | http-outgoing-9 >> GET /maven-repository/commons-io/commons-io/2.7/commons-io-2.7.jar HTTP/1.1
18:10:17,876 | DEBUG | ... | http-outgoing-9 >> Host: localhost:8181
...
```

# settings.xml
org.ops4j.pax.url.mvn.settings - 我们可以明确指定一个遵循Maven Settings XML Schema的XML文档位置。
如果没有指定，那么pax-url-aether按照下面的位置搜索settings文件：
  + ${user.home}/.m2/settings.xml
  + ${maven.home}/conf/settings.xml
  + $M2_HOME/conf/settings.xml
默认情况下，使用的是隐式位置(最可能的是${user.home}/.m2/settings.xml)。为什么当我们使用 org.ops4j.pax.url.mvn.repositories这样的属性时，要指定自定义的settings.xml文件呢？ 有几件事只能在这儿配置：
  + HTTP代理
  + 自定义的HTTP头 当访问特殊的远程仓库时需要添加

# 后备仓库
org.ops4j.pax.url.mvn.useFallbackRepositories - 如果为true, 那么除了指定的远程仓库以外，Aether会总是使用http://repo1.maven.org/maven2 仓库。我偏向于明确的定义Maven中央仓库（如果需要的话），所以最好设置为false。

HTTP代理
从pax-url-aether 2.3.0版本以后，使用http-wagon和httpclient库。HTTP代理在 settings.xml 文件中配置。这里有一个示例：

```xml
<!--    This is the place to configure http proxies used by Aether.    
If there's no proxy for "https" protocol, proxy for "http" will be used when accessing remote repository-->
<proxies>
    <proxy>
        <id>proxy</id>
        <host>127.0.0.1</host>
        <port>3128</port>
        <protocol>http</protocol>
        <username></username>
        <password></password>
        <nonProxyHosts>127.0.0.*|*.repository.corp</nonProxyHosts>
    </proxy>
</proxies>
```

自定义HTTP头
在 org.ops4j.pax.url.mvn.repositories 属性中指定的每一个远程仓库都应该指定一个id=IDENTIFIER 选项(参见更多选项Repository Options)。有了这个标识符，我们就可以配置额外的HTTP头，来访问特殊的仓库。
例如，settings.xml配置:
```xml
<!--    pax-url-aether may use the below configuration to add custom HTTP headers 
when accessing remote repositories    with a given identifier-->
<servers>
    <server>
        <id>maven.central.repo</id>
        <configuration>
            <httpHeaders>
                <httpHeader>
                    <name>User-Agent</name>
                    <value>Karaf</value>
                </httpHeader>
                <httpHeader>
                    <name>Secret-Header</name>
                    <value>secret_value</value>
                </httpHeader>
            </httpHeaders>
        </configuration>
    </server>
</servers>
```
当访问ID=maven.central.repo的仓库时，我们可以看到下面这些日志：
```
17:30:44,590 | DEBUG | ... | http-outgoing-0 >> GET /maven2/commons-io/commons-io/2.7/commons-io-2.7.jar HTTP/1.1
17:30:44,590 | DEBUG | ... | http-outgoing-0 >> Cache-control: no-cache
17:30:44,590 | DEBUG | ... | http-outgoing-0 >> Cache-store: no-store
17:30:44,590 | DEBUG | ... | http-outgoing-0 >> Pragma: no-cache
17:30:44,591 | DEBUG | ... | http-outgoing-0 >> Expires: 0
17:30:44,591 | DEBUG | ... | http-outgoing-0 >> Accept-Encoding: gzip
17:30:44,591 | DEBUG | ... | http-outgoing-0 >> User-Agent: Karaf
17:30:44,591 | DEBUG | ... | http-outgoing-0 >> Secret-Header: secret_value
17:30:44,591 | DEBUG | ... | http-outgoing-0 >> Host: repo1.maven.org
```
# 全局校验策略
org.ops4j.pax.url.mvn.globalChecksumPolicy - 当Aether从远程仓库拉取构件时，它总是尝试下载该构件的SHA1/MD5校验文件。这有可能会失败。如果仓库的URI没有指定校验策略，那么就会使用这个全局的校验策略。实际上如果指定了全局的校验策略，那么每一个仓库的校验策略就会被忽略。这个属性可能由三个确定的Aether行为:
  + fail - 解析失败
  + warn - 打印WARN信息
  + ignore - 什么都不做
没有办法阻止提取checksum。

# 全局更新策略

org.ops4j.pax.url.mvn.globalUpdatePolicy - 当Aether拉取SNAPSHOT构件时，它需要首先拉取 maven-metadata.xml 。在访问每一个 org.ops4j.pax.url.mvn.repositories 中的远程仓库时，Aether会检查位于org.ops4j.pax.url.mvn.localRepository的 resolver-status.properties 文件是否存在(这个状态文件是特定于给定的groupId, artifactId和version的，例如: <REPOSITORY>/commons-io/commons-io/2.5-SNAPSHOT/resolver-status.properties).。我们可以控制Aether实际是否应该刷新metadata信息：
  + always - 当解析SNAPSHOT构件时，Aether总是拉取maven-metadata.xml
  + never - 与上面的相反
  + daily - 如果从上一次写入maven-metadata-ID_OF_REPOSITORY.xml.lastUpdated的时间戳（位于文件 resolver-status.properties ）算起过去了一天，那么Aether就拉取maven-metadata.xml。
  + interval:<NUMBER_OF_MINUTES> - 如果给定的分钟数过去了，那么Aether就拉取 maven-metadata.xml 。


# 当本地仓库当成远程仓库

org.ops4j.pax.url.mvn.defaultLocalRepoAsRemote - 这个属性决定是否应该将 org.ops4j.pax.url.mvn.localRepository 属性指定的本地仓库作为第一个远程仓库插入到org.ops4j.pax.url.mvn.localRepository指定的远程仓库列表中。这不是一个好主意。最佳做法应该使用org.ops4j.pax.url.mvn.defaultRepositories。


#  Socket配置

从pax-url-aether 2.5.0版本开始，有一些配置低级别的socket的选项。Apache httpclient库底层使用的是Sockets (阻塞式I/O) 。这里有一个自顶向下的层次列表，用于Java socket实际访问远程仓库：
  + pax-url-aether
  + Eclipse Aether
  + Maven http-wagon
  + Apache httpclient
  + java.net Sockets

下面是可能使用的选项：
  + org.ops4j.pax.url.mvn.socket.readTimeout - 配置阻塞IO的读取超时时间
  + org.ops4j.pax.url.mvn.socket.connectionTimeout - 配置TCP连接建立超时时间
  + org.ops4j.pax.url.mvn.socket.linger - SO_LINGER选项, 默认为-1
  + org.ops4j.pax.url.mvn.socket.reuseAddress - SO_REU SEADDDR选项，默认为false
  + org.ops4j.pax.url.mvn.socket.tcpNoDelay - TCP_NODELAY选项，默认为true
  + org.ops4j.pax.url.mvn.socket.keepAlive - SO_KEEPALIVE 选项，默认为false
  + org.ops4j.pax.url.mvn.connection.bufferSize - buffer大小(读和写) 当访问远程仓库时，httpclient使用的buffer大小，默认为8192
  + org.ops4j.pax.url.mvn.connection.retryCount - Apache httpclient的 DefaultHttpRequestRetryHandler 配置，默认为3


