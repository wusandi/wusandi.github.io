---
layout: post
title:  "Equinox扩展URL"
categories: OSGi
tags:  Equinox URL
author: WuSanDi
---

* content
{:toc}





# 一、扩展java.net.URL之URLStreamHandlerFactory
org.eclipse.osgi.internal.url.EquinoxFactoryManager#installURLStreamHandlerFactory方法向java.net.URL设置了java.net.URLStreamHandlerFactory，它的实现类为org.eclipse.osgi.internal.url.URLStreamHandlerFactoryImpl

当实例化URL时，URLStreamHandlerFactoryImpl创建的URLStreamHandler实例的实现为org.eclipse.osgi.internal.url.URLStreamHandlerProxy，它有一个ServiceTracker字段,用于跟踪org.osgi.service.url.URLStreamHandlerService服务。它会获取服务属性"url.handler.protocol"（org.osgi.service.url.URLConstants.URL_HANDLER_PROTOCOL），即URLStreamHandlerService实现类支持的协议类型，并与URLStreamHandlerProxy中的protocol进行比较，若相同，则获取该服务实例，并保存到URLStreamHandlerService realHandlerService字段中，当客户端代码调用URLStreamHandlerProxy openConnection时，会将请求委派给realHandlerService，即真正提供URLConnection实现的类实例。


# 二、扩展java.net.URLConnection之ContentHandlerFactory
org.eclipse.osgi.internal.url.EquinoxFactoryManager#installContentHandlerFactory方法向java.net.URLConnection设置了java.net.ContentHandlerFactory，它的实现类为org.eclipse.osgi.internal.url.ContentHandlerFactoryImpl

# 三、URL实例化并openConnetion流程

## 1、URL url = new URL(string)
![avatar](https://raw.githubusercontent.com/wusandi/wusandi.github.io/master/img/equinox-url/OSGI-URL1.png)

## 2、url.openConnection()
![avatar](https://raw.githubusercontent.com/wusandi/wusandi.github.io/master/img/equinox-url/OSGI-URL2.png)

## 3、url.openStream()访问ops4j的mvn协议
![avatar](https://raw.githubusercontent.com/wusandi/wusandi.github.io/master/img/equinox-url/OSGI-URL3.png)

