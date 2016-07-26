---
layout: post
title: "Hadoop 2.7.2编译"
date: 2016-07-20 00:11:12
author: 伊布
categories: tech
tags: hadoop
cover:  "/assets/instacode.png"
---

调试Spark支持S3作为DataSource的时候，需要改下hadoop-aws这个包，所以编译了一把Hadoop，具体步骤可以参考[这个韩国人的文章](http://pl.postech.ac.kr/~maidinh/blog/?p=170)，包括centos和ubuntu两个操作系统的说明。注意Hadoop编译需要cmake/make/protoc/libssl等。我的操作系统是ubuntu，直接照抄下面即可（相对韩国人我删掉了 -Drequire.snappy参数）。


```bash
# sudo apt-get install maven libssl-dev build-essential pkgconf cmake libprotobuf8 protobuf-compiler
# tar xvf hadoop-2.7.2-src.tar.gz
# cd hadoop-2.7.2-src
# mvn package -Pdist,native -DskipTests -Dtar -Dmaven.javadoc.skip=true -Drequire.openssl
```

编译出来的hadoop-2.7.2.tar.gz包在{hadoopBaseDir}/hadoop-dist/target，解压配置继续。如果是升级，可以只拷贝升级的包到工作目录。

不过在编译hadoop的时候遇到了一个比较特别的问题，记录下。

Maven的出错信息如下：


```bash
org.apache.hadoop.util.NativeCrc32 org.apache.hadoop.net.unix.DomainSocket org.apache.hadoop.net.unix.DomainSocketWatcher
[INFO]
[INFO] --- maven-antrun-plugin:1.7:run (make) @ hadoop-common ---
[INFO] Executing tasks

main:
     [exec] -- The C compiler identification is GNU 4.8.4
     [exec] -- The CXX compiler identification is GNU 4.8.4
     [exec] -- Check for working C compiler: /usr/bin/cc
     [exec] -- Check for working C compiler: /usr/bin/cc -- works
     [exec] -- Detecting C compiler ABI info
     [exec] -- Detecting C compiler ABI info - done
     [exec] -- Check for working CXX compiler: /usr/bin/c++
     [exec] -- Check for working CXX compiler: /usr/bin/c++ -- works
     [exec] -- Detecting CXX compiler ABI info
     [exec] JAVA_HOME=, JAVA_JVM_LIBRARY=JAVA_JVM_LIBRARY-NOTFOUND
     [exec] JAVA_INCLUDE_PATH=JAVA_INCLUDE_PATH-NOTFOUND, JAVA_INCLUDE_PATH2=JAVA_INCLUDE_PATH2-NOTFOUND
     [exec] CMake Error at JNIFlags.cmake:120 (MESSAGE):
     [exec]   Failed to find a viable JVM installation under JAVA_HOME.
     [exec] Call Stack (most recent call first):
     [exec]   CMakeLists.txt:24 (include)
     [exec] 
     [exec] 
     [exec] -- Detecting CXX compiler ABI info - done
     [exec] -- Configuring incomplete, errors occurred!
     [exec] See also "/home/dtdream/hadoop/hadoop-2.7.2-src/hadoop-common-project/hadoop-common/target/native/CMakeFiles/CMakeOutput.log".
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
```
Google了半天，基本都是非常肯定的说没有安装libssl-dev、zlib1g-dev之类[缺包的问题](http://stackoverflow.com/questions/23340101/hadoop-2-4-failed-to-execute-goal-org-apache-maven-pluginsmaven-antrun-plugin1)，但很显然我是装了的。后来找到一个遇到相同问题的[文章](https://wp-midsum.rhcloud.com/?p=37)，提到了这个问题的原因是设置JAVA_HOME不符合Hadoop的需求。

由于我的操作系统上装了三套JDK(2套1.7一套1.8)，所以JAVA_HOME设置的值是`/usr`，这种情况下Hadoop/Spark/Hbase/Hive都跑的好好的。但Hadoop编译需要设置到具体的JDK，在我这里就是`export JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk-amd64`。那么是什么原因导致的呢？

找到上面报错的{hadoopBaseDir}/hadoop-common-project/hadoop-common/src/JNIFlags.cmake这个文件，看错误是JAVA_JVM_LIBRARY变量为空。走读下这个cmake文件，JAVA_JVM_LIBRARY/JAVA_INCLUDE_PATH2/JAVA_INCLUDE_PATH这几个变量都是从变量_JDK_DIRS取的值，而这个值是从JAVA_HOME系统变量中**直接**添加后缀`/jre/lib`什么的，而这个目录在/usr里不存在，只存在于具体的JDK路径中。

```
    FILE(TO_CMAKE_PATH "$ENV{JAVA_HOME}" _JAVA_HOME)
..
    SET(_JDK_DIRS "${_JAVA_HOME}/jre/lib/${_java_libarch}/*"
                  "${_JAVA_HOME}/jre/lib/${_java_libarch}"
                  "${_JAVA_HOME}/jre/lib/*"
                  "${_JAVA_HOME}/jre/lib"
                  "${_JAVA_HOME}/lib/*"
                  "${_JAVA_HOME}/lib"
                  "${_JAVA_HOME}/include/*"
                  "${_JAVA_HOME}/include"
                  "${_JAVA_HOME}"
    )
    FIND_PATH(JAVA_INCLUDE_PATH
        NAMES jni.h
        PATHS ${_JDK_DIRS}
        NO_DEFAULT_PATH)
    #In IBM java, it's jniport.h instead of jni_md.h
    FIND_PATH(JAVA_INCLUDE_PATH2
        NAMES jni_md.h jniport.h
        PATHS ${_JDK_DIRS}
        NO_DEFAULT_PATH)
    SET(JNI_INCLUDE_DIRS ${JAVA_INCLUDE_PATH} ${JAVA_INCLUDE_PATH2})
    FIND_LIBRARY(JAVA_JVM_LIBRARY
        NAMES jvm JavaVM
        PATHS ${_JDK_DIRS}
        NO_DEFAULT_PATH)
    SET(JNI_LIBRARIES ${JAVA_JVM_LIBRARY})
    MESSAGE("JAVA_HOME=${JAVA_HOME}, JAVA_JVM_LIBRARY=${JAVA_JVM_LIBRARY}")
    MESSAGE("JAVA_INCLUDE_PATH=${JAVA_INCLUDE_PATH}, JAVA_INCLUDE_PATH2=${JAVA_INCLUDE_PATH2}")
    IF(JAVA_JVM_LIBRARY AND JAVA_INCLUDE_PATH AND JAVA_INCLUDE_PATH2)
        MESSAGE("Located all JNI components successfully.")
    ELSE()
        MESSAGE(FATAL_ERROR "Failed to find a viable JVM installation under JAVA_HOME.")
    ENDIF()
```

据说HBASE可以编译成功。

另1，贴下ubuntu一套系统下切换JDK的命令。

```
sudo update-alternatives --config javac
```

根据提示选择1/2/3即可。

另2，默认ubuntu 14.04 server不能安装OpenJDK 8，如何安装可以看这篇[文章](http://ubuntuhandbook.org/index.php/2015/01/install-openjdk-8-ubuntu-14-04-12-04-lts/)的说明。





