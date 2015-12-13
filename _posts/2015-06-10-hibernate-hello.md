---
layout: post
title: "Hibernate之Hello World"
date: 2015-06-10 19:22:01
tags: hibernate
author: 伊布
categories: tech
cover:  "/assets/instacode.png"
---


Hibernate是一个优秀的持久化框架，其主要功能是将内存里的瞬时状态，通过JDBC持久化到硬盘数据库上。Hibernate以面向对象的思想来解决数据库的问题，可以简化数据库的访问。
这篇文章通过一个简单的示例，来建立Hibernate的初步认识，较水，记录用。

示例代码基本是从这篇[文章](http://www.javaweb.cc/architecture/hibernate/132325.shtml)里抄来的。

首先，得有一个~~女朋友~~表。我在mytest数据库里创建了一个student表。
```sql
CREATE TABLE `student` (
  `id` int(11) NOT NULL auto_increment,
  `name` varchar(255) default NULL,
  `password` varchar(255) default NULL,
  PRIMARY KEY  (`id`)
) ;
```

###1、创建工程，导入Hibernate库。
我下载的版本是4.3.10，从[官方网站](http://hibernate.org/orm/downloads/)上下载下来的是Hibernate ORM（hibernate-release-4.3.10.Final.zip）。要导入的lib文件在`.\hibernate-release-4.3.10.Final\lib\required\`，其他的先不管（我也不知道干啥用的）。
想不起来当时怎么把lib导入到IDEA了，现在导入没有图标了。

###2、建立包`com.xxx.hibernatest`

###3、建立实体类的映射
在hibernatest包里新建Student.hbm.xml文件，编写对象关系映射文件，把对象关系映射的逻辑放在此处，这个文件包括表和字段的对象关系，当操作对象时，该文件通过java反射机制产生的方法，会把对象的方法转为关系的方法。

```xml
<?xml version="1.0"?>
<!DOCTYPE hibernate-mapping PUBLIC
        "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
        "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">

<hibernate-mapping package="com.dtdream.hibernatest">

    <class name="Student" table="student">
        <id name="id">
            <generator class="native"/>
        </id>
        <property name="name"/>
        <property name="age"/>
    </class>

</hibernate-mapping>
```

###4、对象类定义。
在hibernatest包里新建Student.java。它对应于数据库的一个具体的表，需要定义数据库的列、列方法等，实际就是用Java来描述表。

```java
package com.dtdream.hibernatest;

public class Student {
    private int id;
    private String name;
    private  int age;
    public int getId() {
        return id;
    }
    public String getName() {
        return name;
    }
    public int getAge() {
        return age;
    }
    public void setId(int id) {
        this.id = id;
    }
    public void setName(String name) {
        this.name = name;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```

###5、main入口
这里可以看到Hibernate简单的用法，代码看一看很清楚，就不多说了，名字随便起。

```java
package com.dtdream.hibernatest;

import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;

public class StudentTest {
    public static void main(String[] args) {
        Student s = new Student();
        //s.setId();	这句不需要，我们创建的数据库是increasement的
        s.setAge(23);
        s.setName("dtd");

        Configuration cfg = new Configuration();
        SessionFactory sf = cfg.configure().buildSessionFactory();
        Session session = sf.openSession();
        session.beginTransaction();
        session.save(s);
        session.getTransaction().commit();
        session.close();
        sf.close();
    }
}
```

###6、Hibernate主配置
前面我们有了数据库操作主流程，有了对象关系映射文件，有了表的对象，表在前面也创建了，那么就剩下怎么访问数据库了：地址、端口号、数据库名、driver。
这里提供一个`hibernate.cfg.xml`的数据库配置文件。配置文件放到工程的根目录，不要放到包里。
Hibernate实际是封装了JDBC，需要对应数据库的driver lib，对端数据库用的是mysql，所以需要把mysql-connector-java-5.1.35-bin也加到project的lib库里。
配置文件就不说了，照着葫芦画瓢就行。

```xml
<?xml version='1.0' encoding='utf-8'?>
<!DOCTYPE hibernate-configuration PUBLIC
        "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
        "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
<session-factory>
    <!-- Database connection settings -->
    <property name="connection.driver_class">com.mysql.jdbc.Driver</property>
    <property name="connection.url">jdbc:mysql://192.168.5.29:3306/mytest</property>
    <property name="connection.username">root</property>
    <property name="connection.password"></property>

    <!-- JDBC connection pool (use the built-in) -->
    <property name="connection.pool_size">1</property>
    <!-- SQL dialect -->
    <property name="dialect">org.hibernate.dialect.MySQLDialect</property>
    <!-- Enable Hibernate's automatic session context management -->
    <property name="current_session_context_class">thread</property>
    <!-- Disable the second-level cache  -->
    <property name="cache.provider_class">org.hibernate.cache.internal.NoCacheProvider</property>
    <!-- Echo all executed SQL to stdout -->
    <property name="show_sql">true</property>
    <!-- Drop and re-create the database schema on startup -->
    <property name="hbm2ddl.auto">update</property>
    <mapping resource="com/dtdream/hibernatest/Student.hbm.xml"/>
</session-factory>
</hibernate-configuration>
```

好了，有了上面4个文件，一个简单的Hibernate示例就完成了，run main文件，我们去看mysql数据库里，可以看到添加了一条记录：

```sql
mysql> select * from student;
+----+--------+----------+------+
| id | name   | password | age  |
+----+--------+----------+------+
|  1 | dtd    | NULL     |   23 |
+----+--------+----------+------+
1 rows in set (0.01 sec)
```

再运行一次，可以看到又添加了一条记录：

```bash
mysql> select * from student;
+----+--------+----------+------+
| id | name   | password | age  |
+----+--------+----------+------+
|  1 | dtd    | NULL     |   23 |
|  2 | dtd    | NULL     |   23 |
+----+--------+----------+------+
2 rows in set (0.01 sec)
```

至此，示例结束，以后有机会用Hibernate做一些实际的事情再来分享更深入的东西。
