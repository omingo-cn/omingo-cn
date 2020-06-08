---
title: keycloak-mybatis整合
date: 2020-06-08 14:39:23
categories: Keycloak
tags:
- keycloak
---

# keycloak-整合mybatis

为方便一些不会jpa的同学，特意整理了一下与mybatis的结合使用方式。

放码过来。
## 依赖\配置
数据源配置参考： [keycloak-多数据源配置](https://www.omingo.cn/2020/05/11/keycloak-多数据源配置/)
maven 依赖和打包配置。
`pom.xml`
```xml
<!--...-->
    <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis</artifactId>
      <version>3.5.4</version>
    </dependency>
<!--...-->
 <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-war-plugin</artifactId>
        <version>3.2.3</version>
        <configuration>
          <failOnMissingWebXml>false</failOnMissingWebXml>

          <webResources>
            <resource>
              <directory>${project.build.directory}/classes/META-INF/services</directory>
              <targetPath>META-INF/services</targetPath>
              <includes>
                <include>*.*</include>
              </includes>
            </resource>
            <resource>
              <directory>src/main/java</directory>
              <targetPath>WEB-INF/classes</targetPath>
              <includes>
                <include>**/*Mapper.xml</include><!--关键点-->
              </includes>
              <filtering>true</filtering>
            </resource>
          </webResources>
        </configuration>
      </plugin>
     <!--...--> 
```

`resources/META-INF`加入`mybatisconfig.xml`配置文件：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
  <environments default="development">
    <environment id="development">
      <transactionManager type="MANAGED"/>
      <dataSource type="JNDI">
        <property name="data_source" value="java:jboss/datasources/DapengXADS"/><!--自定义数据源名称要一致-->
      </dataSource>
    </environment>
  </environments>
  <mappers>
    <mapper resource="com/dapeng/cloud/models/mybatis/UserMapper.xml"/>
  </mappers>
</configuration>

```


## UserStorageProviderFactory

```java
@JBossLog
@AutoService(MybatisStorageProviderFactory.class)
public class MybatisStorageProviderFactory implements UserStorageProviderFactory<DapengUserStorageProvider>{
  private static final String PROVIDER_NAME="dapeng-jdbc";
  private static final String MYBATIS_CONFIG="META-INF/mybatis-config.xml";
  private SqlSessionFactory sqlSessionFactory;

  @Override
  public void init(Scope config) {
    try {
      InputStream in = Resources.getResourceAsStream(MYBATIS_CONFIG);
      sqlSessionFactory=new SqlSessionFactoryBuilder().build(in);
    } catch (IOException e) {
      log.error("创建sqlSessionFactory失败",e);
      e.printStackTrace();
    }

  }

  public String getId() {
    return PROVIDER_NAME;
  }

  public MybatisUserStorageProvider create(KeycloakSession keycloakSession, ComponentModel componentModel) {

      return new MybatisUserStorageProvider(keycloakSession,componentModel,sqlSessionFactory.openSession());
  }

}

```

## MybatisUserStorageProvider

```java
@JBossLog
@Data
@AllArgsConstructor
public class MybatisUserStorageProvider implements UserStorageProvider, UserLookupProvider,
  CredentialInputValidator {
  private  KeycloakSession session;
  private  ComponentModel componentModel;
  private SqlSession sqlSession;

  @Override
  public UserModel getUserById(String id,
    RealmModel realmModel) {
    User user = (User)sqlSession.selectOne("com.dapeng.cloud.models.mybatis.UserMapper.findById", id);
    log.debugv("用户:{0}",user);
    return new UserAdapter(session,realmModel,componentModel,user);
  }

  @Override
  public UserModel getUserByUsername(String username, RealmModel realmModel) {
    log.debugv("获取用户信息 用户名:{0},域:{1}",username,realmModel.getName());
    User user =(User)sqlSession.selectOne("com.dapeng.cloud.models.mybatis.UserMapper.findByMobile", username);

    return new UserAdapter(session,realmModel, componentModel,user);
  }

  @Override
  public UserModel getUserByEmail(String s, RealmModel realmModel) {
    return null;
  }

  @Override
  public boolean supportsCredentialType(String credentialType) {
    return PasswordCredentialModel.TYPE.equals(credentialType);
  }

  @Override
  public boolean isConfiguredFor(RealmModel realm, UserModel user, String credentialType) {
    return PasswordCredentialModel.TYPE.equals(credentialType)&&user instanceof UserAdapter;
  }

  @Override
  public boolean isValid(RealmModel realm, UserModel user,
    CredentialInput credentialInput) {
    if (!supportsCredentialType(credentialInput.getType())) return false;
    String in = credentialInput.getChallengeResponse();
    log.infof("input password: %s ,", in);

    return "123456".equals(in);
  }

  @Override
  public void close() {

  }
}

```

剩下的就是mybatis的常规操作了，这里不在赘述。

有问题请留言。