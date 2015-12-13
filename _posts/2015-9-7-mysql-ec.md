---
layout: post
title: "mysql:C源代码内嵌SQL语句的预编译工具"
date: 2015-09-07 19:58:40
author: 伊布
categories: tech
tags: mysql
cover:  "/assets/instacode.png"
---


---
**本文较水  哪篇不水**

今天遇到一个比较好玩的东西，记下来跟大家分享。
我们知道操作数据库，在JAVA环境中一般会使用JDBC，不管是mysql、oracle还是别的什么数据库都可以比较好的支持；但在一些比较古老（或者说比较谨慎）的系统中，比如银行业，可能其应用使用的还是C语言写的，这种情况JDBC是没门了，只能调用对应数据库提供的C lib，例如mysql的C库提供了这些接口：

```c
MYSQL *mysql_init(MYSQL *mysql)
MYSQL *mysql_real_connect(MYSQL *mysql, const char *host, const char *user, const char *passwd, const char *db, unsigned int port, const char *unix_socket, unsigned long client_flag)
int mysql_query(MYSQL *mysql, const char *stmt_str)
```

有了C库，按照一般的思路，直接在C里面调用对应的API就好了，但显然这样不如直接写SQL方便。于是像Oracle就提供了[Pro*C](http://baike.baidu.com/link?url=GeQqyf0TdG0m3EWrFIVO7LIw8yUsjzAFA7E4VKm_s21kOWD2rePSmVCKCyUWT5GGdKm-v8EJ8U0HWZMZGmlv8K)这种工具。
说白了PRO\*C就是一种预编译工具，我们写的代码是C和SQL混合的，这样的代码没办法直接编译，gcc不认。PRO*C会将该代码（*.pc）转为纯C代码，这样就可以用gcc编译为可执行文件了。但PRO*C是为ORACLE服务的，编译出来的代码调用的也是ORACLE的C库，显然我们想给mysql用是行不通的，需要一款类似的工具。其他的数据库，如IBM DB2有PREP，Informix和Postgre SQL Server都有自己的ESQL/C工具。

在网上扒了半天，找到了这么一个工具，[open esql](http://sourceforge.net/projects/open-esql/)。下载下来解压后看到两个文件夹，e2odbc是使用ODBC，理论上应该适应所有类型的数据库，而e2mysql是我们要用的。进到e2mysql看到有三个文件：`e2mysql.awk  e2mysql.cpp  e2mysql.h`。e2mysql.awk是一个awk脚本，用来预处理pc混合文件；e2mysql.cpp用来定义数据库连接信息。

使用步骤如下：

**1、使用awk脚本转换pc文件为C++文件。**

```bash
# awk -f e2mysql.awk xxx.pc > xxx.cpp
```

预处理实际就是文本转换处理，正是awk的用武之地。

**2、将xxx.cpp与e2mysql.cpp一起编译。**

```bash
# g++ xxx.cpp e2mysql.cpp -o xxx `mysql_config --cflags --libs`
```

步骤很简单，但过程中遇到了一些问题，这里记一下，您自己解决起来应该也比较简单。
1、g++编译的时候提示没有sqlda.h、sqlcpr.h。

```c
xxx.cpp:7:21: error: sqlda.h: No such file or directory
xxx.cpp:8:20: warning: extra tokens at end of #include directive
xxx.cpp:8:22: error: sqlcpr.h: No such file or directory
```

PRO*C的源文件里会调用这两个头文件，我们没有，删掉即可。如果是在生产环境，建议创建这两个头文件，为空即可。

2、提示e2mysql.h里没有定义strlen。

```c
e2mysql.h: In function ‘int32_t SQLBindParmPoly(std::string&, const char*, char, uint16_t)’:
e2mysql.h:176: error: ‘strlen’ was not declared in this scope
```

strlen是C的string函数，需要包含cstring头文件：`#include <cstring>`。

3、提示e2mysql.h里没有定义atoi。

```c
e2mysql.h: In function ‘int32_t SQLBindColPoly(const char*, int16_t&, uint16_t)’:
e2mysql.h:319: error: ‘atoi’ was not declared in this scope
```

atoi是stdlib里的函数，需要包含该头文件：`#include <stdlib.h>`。

4、提示EXEC不识别。

```
xxx.cpp:13: error: ‘EXEC’ does not name a type
```

找到代码里是这一句：

```sql
/*RELEASE_CURSOR=YES 使PROC 在执行完后释放与嵌入SQL有关资源*/
EXEC ORACLE OPTION (RELEASE_CURSOR = YES);
```

显然这句是跟ORACLE相关的，工具不认。我这里粗暴的将这句注释掉了。

5、提示使用了int16类型不识别。

```
xxx.cpp:33: error: ‘int16’ does not name a type
xxx.cpp:61: error: ‘int16’ does not name a type
xxx.cpp:78: error: ‘int16’ does not name a type
```

应该是作者的环境不是gcc，而是一些能认int16这种变体的windows环境。这个是在awk预编译的时候生成的，需要改awk脚本源码，我这里是整体替换为了short int。

6、提示sqlca_struct里没有sqlerrm。

```
xxx.cpp: In function ‘void sql_error(char*)’:
xxx.cpp:92: error: ‘struct sqlca_struct’ has no member named ‘sqlerrm’
```

如上，sqlca是oracle特有的，这儿不识别，注释掉即可。

7、编译成功，执行的时候提示连不上`'abc.xyz.com' `。
看了下awk脚本，没有对connect做转换，而e2mysql.cpp里有个SQLHelperConnect(void)函数，里面可以修改数据库连接信息，改之，重新编译，验证OK。

```bash
# ./xxx
Connecting to database
Connected: test
c1=100,c2=200,c3=300
```

```bash
mysql> select * from xxx;
+------+------+------+
| c1   | c2   | c3   |
+------+------+------+
|  100 |  200 |  300 |
+------+------+------+
1 row in set (0.00 sec)

```

8、如果提示没有mysql.h，或者找不到mysqlclient的lib，建议安装mysql-devel(centos 6.5)，并且在编译的时候，不要手工指定-I、-L、-lmysqlclient，而是用mysql_config自动配置：`mysql_config --cflags --libs`。


总的来说这个脚本在不复杂的情况下还是能用的，但确实不成熟，如果有大量的C文件，需要一个一个去转，而且每个文件可能都要做一些修改；没能像PRO\*C那样可以返回SQL执行的结果，用户用起来不方便；等等。解决问题的思路没错，但如果要商业化，还是有很长的路要走。

所有源文件放到github上了，参见[open-esql-s](https://github.com/silenceshell/open-esql-s)。
