---
layout: post
title: "Spark（九）：Thrift Server的用户认证"
date: 2016-05-18 08:11:12
author: 伊布
categories: tech
tags: spark
cover:  "/assets/instacode.png"
---

[Hiverserver](https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2#SettingUpHiveServer2-Authentication/SecurityConfiguration)2支持如下四种：

- KERBEROSE
- LDAP
- CUSTOM
- NONE

默认为NONE，即只要用户对即可，不检查密码。
类似Hiveserver2，Spark thrift server(sts)也支持这几种。前面我们已经具备了权限管理(Authorization)的能力，但还需要对用户的identity做认证，先采用比较简单的CUSTOM认证方式，可以参考[MapR的这篇文章](http://doc.mapr.com/display/MapR40x/Using+HiveServer2#UsingHiveServer2-ConfiguringCustomAuthentication)，主要是实现PasswdAuthenticationProvider接口。


```java
public interface PasswdAuthenticationProvider {
    void Authenticate(String var1, String var2) throws AuthenticationException;
}
```


插件比较简单，不过我不太熟悉java，下面会将编码调试过程中遇到的一些问题简要记录下。

### 用户验证

比较简单：JDBC或者beeline登录sts时，输入用户名、密码，sts会将这两个字符串交给自定义插件检查；插件从数据库中查询该用户对应的加密后的密码（我们有一套用户管理系统），然后将输入的密码加密后与数据库查到的密码对比，如果不一致则**抛异常**，认证失败。代码如下。

```java
public class Authentication implements PasswdAuthenticationProvider {
..
    @Override
    public void Authenticate(String userName, String passwd)
            throws AuthenticationException {
        LOG.info("user: " + userName + " try login.");

        String passwdBase = getPasswd(userName);
        if (null == passwdBase) {
            String message = "User not found. user:"+userName;
            LOG.info(message);
            throw new AuthenticationException(message);
        }

        BCryptPasswordEncoder bCryptPasswordEncoder =
                new BCryptPasswordEncoder(10);
        boolean isSuccess = bCryptPasswordEncoder.matches(passwd, passwdBase);
        if (!isSuccess) {
            String message = "User name and password is mismatch. user:"+userName;
            LOG.info(message);
            throw new AuthenticationException(message);
        }
    }
```

密码加密使用了spring-security的core jar，需要将这个jar包放到spark的lib目录下，并在启动thrift server时显式的注明（下文有启动脚本）。
数据库建立连接、查询，因为查询极简单只有一条select，所以我没有用hibernate之类的框架，直接JDBC取的，代码就不贴了。

### 数据库配置解析
在处理数据库连接配置信息时我想和apache家族保持一致使用xml，不过java处理xml实在太繁琐了（相比下python就很简单），好在hadoop提供了Configuration类，专门用以解析hadoop的配置文件，类似下面这种：

```xml
<configuration>
  <property>
    <name>hive.server2.custom.authentication.url</name>
    <value>jdbc:mysql://{ip}/{database}</value>
  </property>

</configuration>

```

对编码非常友好：

```java
    private static boolean parseConf() {
        Configuration conf = new Configuration();
        conf.addResource("hive-custom.xml");
        db_url = conf.get("hive.server2.custom.authentication.url");
```

有关Configuration的详细使用参考[这里](https://hadoop.apache.org/docs/r2.6.1/api/org/apache/hadoop/conf/Configuration.html)，上面的代码里直接add String类型的资源，需要保证这个资源在java进程的classpath中，启动sts时，spark-submit会把{spark-project}/conf加到-cp参数中。也可以使用linux的路径来取得Path变量后将变量加到资源中，只是不如cp灵活。

我新增了个hive-custom.xml文件用于保存用户数据库配置。当然也可以加到hive-site.xml中，但是会污染HiveConf，不推荐。这里需要特别说明下addResource和addDefaultResource，前者可以覆盖之前的配置并保持final不可修改，后者按顺序加载。另外一个很重要的区别是addDefaultResouce是static的，自定义的属性最好不要用（更具体的可以看下hive解析hive-site.xml和hive-default.xml的过程，是在类的静态初始代码中add的）。


### 配置

1、修改{spark_home}/conf/hive-site.xml，增加下面两个配置：

```xml
  <property>
    <name>hive.server2.authentication</name>
    <value>CUSTOM</value>
  </property>

  <property>
    <name>hive.server2.custom.authentication.class</name>
    <value>com.dtdream.spark.Authentication</value>
  </property>
```

其中CUSTOM指定sts使用自定义插件认证，即hive.server2.custom.authentication.class的类。如果不需要认证，将这两个配置删掉即可，默认的认证策略为NONE，即无认证。

2、增加配置文件：{spark_home}/conf/hive-custom.xml，增加如下配置：

```xml
<configuration>
  <property>
    <name>hive.server2.custom.authentication.url</name>
    <value>jdbc:mysql://{ip}/{database}</value>
  </property>

  <property>
    <name>hive.server2.custom.authentication.user</name>
    <value>{db_user}}</value>
  </property>

  <property>
    <name>hive.server2.custom.authentication.password</name>
    <value>{db_password}</value>
  </property>

</configuration>

```


在{spark_home}/lib中增加2个jar包：编译出来的sts_authen-1.0-SNAPSHOT.jar，和spring-security-core-4.0.1.RELEASE.jar。


### 启动

注意指定jar包。thrift server只是driver，不需要用`--jar`参数。

```
./start-thriftserver.sh --master yarn  --executor-memory 6g  --driver-memory 6g --conf spark.yarn.executor.memoryOverhead=4096 --driver-class-path /home/dtdream/apache-hive-1.2.1-bin/lib/mysql-connector-java-5.1.35-bin.jar:/home/dtdream/spark/spark-1.6.1-bin-hadoop2.6/lib/sts_authen-1.0-SNAPSHOT.jar:/home/dtdream/spark/spark-1.6.1-bin-hadoop2.6/lib/spring-security-core-4.0.1.RELEASE.jar
```

