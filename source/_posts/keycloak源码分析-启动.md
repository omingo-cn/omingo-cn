---
title: keycloak源码分析-启动
date: 2020-05-09 18:12:34
categories: Keycloak
tags:
- keycloak

---

分析版本: 10.0.0

## web.xml

![](20200508112409320_335053284.png)

1. `module-name`为*auth*,这就是为什么启动keyloak服务后要通过 http://domain/auth进行访问应用的原因了.

2. 我们发现有一个`load-on-startup`的servlet声明,这个是restEasy的启动声明,详细请参考[Older servlet containers](https://docs.jboss.org/resteasy/docs/4.5.3.Final/userguide/html/Installation_Configuration.html#d4e143).

## WildflyLifecleListener

里面有个`listener`声明,实际上启动阶段这个listener什么都没做,到时容器关闭的时候,执行了一个shutdownHook.

![](20200508115559301_1073305909.png)

## KeycloakApplication

这个是keycloak启动的入口.

```flow

loadConfig=>operation: 加载配置

createSessionFectory=>operation: 创建createSessionFactory并将实例加入servletContext

resteasyContextPush=>operation: 当前keycloakApplication实例加入resteasyContext

addResource=>operation: 添加resource,filter,事务管理,异常处理和资源解析器

addHook=>operation: 执行启动hook,关联shutdownhook

loadConfig->createSessionFectory->resteasyContextPush->addResource->addHook->loadConfig

```

这里的启动hook主要是做一些系统升级相关的逻辑.关联的shutdownhook主要是把之前创建的sessionFactory关掉. 这个shutdownhook实在上一节中提到的`WildflyLifecleListener`中调用的

启动过程中不管出现任何异常,keycloak都会直接exit.

## KeycloakSessionServletFilter

目光再次回到`web.xml`上来,这里还有一个`KeycloakSessionServletFilter`,看一下里面的`doFilter`都做了什么:

![](20200508132807887_775282546.png)SessionFactory和

```flow

encoding=>operation: 设置请求编码为UTF-8

getSessionFactory=>operation: 从servletRequestContext中取出sessionFactory

createSession=>operation: 创建keycloakSession并push到ResteasyContext

connection=>operation: 由request创建了一个ClientConnection对象并关联到sessionContext和push到resteasyContext

tx=>operation: 从session获取事务管理器实例并push进resteasyContext,然后打开事务

doFilter=>operation: doFilter

exception=>condition: 有异常?

rollbackTx=>operation: 回滚事务

closeSession=>operation: 关闭session

cleanRestEasyContext=>operation: 清理restEasyContextData

encoding->getSessionFactory->createSession->connection->tx->doFilter->exception

exception(yes)->closeSession

exception(no)->rollbackTx->closeSession->cleanRestEasyContext

```

这个filter负责创建session,并且打开事务,当出现异常的时候自动回滚事务.当没又出现异常的时候这个事务是如何提交的呢,继续往下分析.

## KeycloakTransactionCommitter

在上面的KeycloakApplication的代码分析中有这么一句`classes.add(KeycloakTransactionCommitter.class);`看名字应该是做事务提交用的.

![](20200508135716767_851192680.png)

果真如此,但是它只负责在response阶段提交事务,如果提交事务出现异常的时候会直接将异常抛出,然后由`KeycloakSessionServletFilter`捕获后做回滚.

## RestEasyContext

目前已经大该分析完了keycloak的启动过程,启动完成后restEasyContext中被push了下面这些对象,这些对象是可以直接通过注解进行注入的.


|  序号   |  类型   |  作用域   |  描述   |
| --- | --- | --- | --- |
|   1  | KeycloakApplication    |  gloable   |     |
|   2 |  KeycloakApplication   |   thread  |  for injection   |
|   3  | KeycloakSession    |   thread  |     |
|   4  |  ClientConnection   |  thread   |     |
|   5  |  KeycloakTransaction   |  thread   |     |