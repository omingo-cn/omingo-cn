---
title: keycloak-多数据源配置
date: 2020-05-11 17:08:40
categories: Keycloak
tags:
- keycloak
---

# keycloak-连接自己的数据库
当开发UserStorage SPI时,我们可能需要访问一个非keycloak数据库来读取用户数据.
本文演示如何给keycloak加一个数据库源.
## 修改keycloak数据库为XA数据源
关于java分布式事务请参考[JTA和XA](https://blog.csdn.net/zhouhao88410234/article/details/91872872?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-8&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-8)

在standalone.xml搜索 urn:jboss:domain:datasources,在datasources标签下增加:
```xml
                <xa-datasource jndi-name="java:jboss/datasources/KeycloakXADS" pool-name="KeycloakXADS">
                    <xa-datasource-property name="url">
                        jdbc:mysql://localhost:3306/keycloak?useSSL=false
                    </xa-datasource-property>
                    <driver>mysql</driver>
                    <security>
                        <user-name>root</user-name>
                        <password>root</password>
                    </security>
                    <validation>
                        <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLValidConnectionChecker"/>
                        <exception-sorter class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLExceptionSorter"/>
                    </validation>
                </xa-datasource>
```
搜索` <spi name="connectionsJpa">` ,然后修改为:
```xml
            <spi name="connectionsJpa">
                <provider name="default" enabled="true">
                    <properties>
                        <property name="dataSource" value="java:jboss/datasources/KeycloakXADS"/><!--修改为XA数据源-->
                        <property name="showSql" value="true"/>
                        <property name="initializeEmpty" value="true"/>
                        <property name="migrationStrategy" value="update"/>
                        <property name="migrationExport" value="${jboss.home.dir}/keycloak-database-update.sql"/>
                    </properties>
                </provider>
```
## 增加自己的XA数据源
在standalone.xml搜索 urn:jboss:domain:datasources,在datasources标签下增加:
```xml
                <xa-datasource jndi-name="java:jboss/datasources/DapengXADS" pool-name="DapengXADS">
                    <xa-datasource-property name="url">
                        jdbc:mysql://192.168.1.254:3306/dapeng_app?useSSL=false
                    </xa-datasource-property>
                    <driver>mysql</driver>
                    <security>
                        <user-name>root</user-name>
                        <password>root</password>
                    </security>
                    <validation>
                        <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLValidConnectionChecker"/>
                        <background-validation>true</background-validation>
                        <exception-sorter class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLExceptionSorter"/>
                    </validation>
                </xa-datasource>
```

##  在自己的SPI Extension中使用新数据源
如何编写 SPI Extension 请参考官方文档 [Service Provider Interfaces (SPI)](https://www.keycloak.org/docs/latest/server_development/#_providers)
首先在 `resources/META-INF/`创建`persistence.xml`:
```xml
<?xml version="1.0" encoding="UTF-8" ?>

<persistence xmlns="http://java.sun.com/xml/ns/persistence"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://java.sun.com/xml/ns/persistence
 http://java.sun.com/xml/ns/persistence/persistence_1_0.xsd" version="1.0">

  <persistence-unit name="keycloak-dapeng" transaction-type="RESOURCE_LOCAL" >
    <description>Dapeng Persistence Unit</description>
    <jta-data-source>java:jboss/datasources/DapengXADS</jta-data-source>

    <class>com.dapeng.cloud.models.jpa.User</class>

    <properties>
      <property name="jboss.entity.manager.factory.jndi.name" value="java:jboss/emf/Dapeng"/><!--暴露为wildfly全局jndi-->
      <property name="hibernate.connection.provider_disables_autocommit" value="true"/><!--注意这里设置,否则出现无法更新数据库错误-->
      <property name="hibernate.hbm2ddl.auto" value="none"/><!--注意这里设置,否则容易删库!!!-->
      <property name="hibernate.show_sql" value="true" />
    </properties>

  </persistence-unit>

</persistence>

>  hibernate.connection.provider_disables_autocommit 不配置为true的话，在进行更新操作的时候会报：`you cannot set autocommit during a managed transaction` 异常。

```
参考资料: [暴露emf全局jndi](https://docs.jboss.org/ejb3/app-server/reference/build/reference/en/html/entityconfig.html#referencing)

编写一个工具类用来获取`EntityManagerFactory`:
```java
public class JndiEntityManagerLookup {
  public static EntityManager getEntityManager(String entityManagerFactoryJndiName) {
    EntityManagerFactory factory = null;

    try {
      factory = (EntityManagerFactory)(new InitialContext()).lookup(entityManagerFactoryJndiName);
    } catch (NamingException var4) {
      throw new RuntimeException(var4);
    }
    return factory.createEntityManager();
  }
}

```
然后在你的`ProviderFactory.create`中就可以愉快的使用新加的数据源了,剩下的请自由发挥:
```java
  public DapengUserStorageProvider create(KeycloakSession keycloakSession, ComponentModel componentModel) {
    EntityManager em=JndiEntityManagerLookup.getEntityManager(EMC_JNDI);
      return new DapengUserStorageProvider(keycloakSession,componentModel,em);
  }
```
