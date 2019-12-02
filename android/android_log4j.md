# Android中使用Log4j及配置说明

2016-06-03 09:09:00   更多



版权声明：本文为博主原创文章，遵循[ CC 4.0 BY-SA ](http://creativecommons.org/licenses/by-sa/4.0/)版权协议，转载请附上原文出处链接和本声明。本文链接：https://blog.csdn.net/zjclugger/article/details/51576156

目前在进行Android开发时使用到了log4j，现在对其配置进行记录。

1. android-logging-log4j 下载地址

https://code.google.com/archive/p/android-logging-log4j/

2. 所依赖的apache的log4j库的下载地址

http://logging.apache.org/log4j/1.2/download.html



Log4j配置说明

## 1.Log4j简介

​    Log4j是Apache的一个开源项目，它允许开发者以任意间隔输出日志信息。主要由以下三大类组件构成：
   （1）Logger---负责输出日志信息，并能够对日志信息进行分类筛选，即决定哪些日志信息应该被输出，哪些该被忽略。

​            Loggers组件输出日志信息时分为5个级别：DEBUG、INFO、WARN、ERROR、FATAL。这五个级别的顺序是：DEBUG<INFO<WARN<ERROR<FATAL。

​            例如，设置某个Logger组件的级别是WARN，那么则只有级别比WARN高的日志信息才能输出，即DEBUG,INFO不会被输出。

​            Logger是有继承关系的，最上层是rootLogger，定义的其他Logger都会继承rootLogger。
 
  （2）Appender---定义了日志输出目的地，指定日志信息应该被输出到什么地方。输出的目的地可以是控制台、文件或网络设备。
 
  （3）Layout---通过在Appender的后面附加Layout来实现格式化输出。一个Logger可以有多个Appender，每个Appender对应一个Layout。



## 2.Loggers

Logger的定义格式：log4j.[loggername]=[level],appenderName,appenderName,…

这里level是指Logger的优先级，appenderName是日志信息的输出地，可以同时定义多个输出地。



## 3.Appenders

Appender的定义格式：

log4j.appender.appenderName = fully.qualified.name.of.appender.class  

// "fully.qualified.name.of.appender.class" 可以指定下面五个目的地中的一个：

Appender类及其作用列表

| Appender类名                              | 作 用                                  |
| ----------------------------------------- | -------------------------------------- |
| org.apache.log4j.ConsoleAppender          | 将日志输出到控制台                     |
| org.apache.log4j.FileAppender             | 将日志输出到文件                       |
| org.apache.log4j.DailyRollingFileAppender | 每天产生一个日志文件                   |
| org.apache.log4j.RollingFileAppender      | 文件大小到达指定尺寸时产生一个新的文件 |
| org.apache.log4j. WriterAppender          | 将日志信息以流格式发送到任意指定的地方 |


（1） ConsoleAppender选项-Threshold=WARN:指定日志消息的输出最低层次。-ImmediateFlush=true:默认值是true,意谓着所有的消息都会被立即输出。-Target=System.err：默认情况下是：System.out,指定输出控制台。





























## 4.Layouts

Layout的定义格式：

部分一log4j.appender.appenderName.layout = fully.qualified.name.of.layout.class

​              //"fully.qualified.name.of.layout.class" 可以指定下面4个格式中的一个：

Layout类及其作用列表

| Layout类名                     | 作 用                                  |
| ------------------------------ | -------------------------------------- |
| org.apache.log4j.HTMLLayout    | 以HTML表格形式布局                     |
| org.apache.log4j.PatternLayout | 可以灵活地指定布局模式                 |
| org.apache.log4j.SimpleLayout  | 包含日志信息的级别和信息字符串         |
| org.apache.log4j.TTCCLayout    | 包含日志产生的时间、线程、类别等等信息 |

（1）HTMLLayout 选项











**日志信息格式中符号所代表的含义**

（1）－X号: X信息输出时左对齐。
（2）%p: 输出日志信息优先级，即DEBUG，INFO，WARN，ERROR，FATAL。
（3）%d: 输出日志时间点的日期或时间，默认格式为ISO8601，也可以在其后指定格式，比如：%d{yyy MMM dd HH:mm:ss,SSS}，输出类似：2002年10月18日 22：10：28，921。
（4）%r: 输出自应用启动到输出该log信息耗费的毫秒数。
（5）%c: 输出日志信息所属的类目，通常就是所在类的全名。
（6）%t: 输出产生该日志事件的线程名。
（7）%l: 输出日志事件的发生位置，相当于%C.%M(%F:%L)的组合,包括类目名、发生的线程，以及在代码中的行数。举例：Testlog4.main(TestLog4.java:10)。
（8）%x: 输出和当前线程相关联的NDC(嵌套诊断环境),尤其用到像java servlets这样的多客户多线程的应用中。
（9）%%: 输出一个"%"字符。
（10）%F: 输出日志消息产生时所在的文件名称。
（11）%L: 输出代码中的行号。
（12）%m: 输出代码中指定的消息,产生的日志具体信息。
（13）%n: 输出一个回车换行符，Windows平台为"\r\n"，Unix平台为"\n"输出日志信息换行。
**可以在%与模式字符之间加上修饰符来控制其最小宽度、最大宽度、和文本的对齐方式。如：**

(1)%20c：指定输出category的名称，最小的宽度是20，如果category的名称小于20的话，默认的情况下右对齐。
(2)%-20c:指定输出category的名称，最小的宽度是20，如果category的名称小于20的话，"-"号指定左对齐。
(3)%.30c:指定输出category的名称，最大的宽度是30，如果category的名称大于30的话，就会将左边多出的字符截掉，但小于30的话也不会有空格。
(4)%20.30c:如果category的名称小于20就补空格，并且右对齐，如果其名称长于30字符，就从左边交远销出的字符截掉。


在Android中使用Log4j

1. 引入log4j-1.2.17.jar

2. 引入android-logging-log4j-1.0.3.jar，其中我下载了这个包的源码，并对其进行了修改，增加了可以设置每天生成一个新日志文件的功能（默认是根据设置的文件最大值来生成新文件的）

部分代码如下：

```java
package com.reconova.communicate.utils;

import android.os.Environment;
import android.os.Process;

import org.apache.log4j.Level;
import org.apache.log4j.Logger;

import de.mindpipe.android.logging.log4j.LogConfigurator;

public class MLog {
    private static Logger logger;
    private static final long MAX_SIZE = 1024 * 1024L;// 1M

    static {
        configLog(Environment.getExternalStorageDirectory() + "/RecoGateData/RecoLogger/Log/", "reco_log", true);
        logger = Logger.getRootLogger();
    }

    public static void v(String tag, String msg) {
        logger.debug(tag + ": " + msg);
    }

    public static void d(String tag, String msg) {
        logger.debug(tag + ": " + msg);
    }

    public static void i(String tag, String msg) {
        logger.info(tag + ": " + msg);
    }

    public static void w(String tag, String msg) {
        logger.warn(tag + ": " + msg);
    }

    public static void e(String tag, String msg) {
        logger.error(tag + ": " + msg);
    }

    public static void e(String tag, String msg, Throwable tr) {
        logger.error(tag + ": " + msg, tr);
    }

    /**
     * 生成日志对象
     *
     * @param filePath 日志输出路径
     * @param fileName 日志文件名
     * @param flag     true:在已存在log文件后面追加 false:新log覆盖以前的log
     */
    public static void configLog(String filePath, String fileName, boolean flag) {
        LogConfigurator logConfigurator = new LogConfigurator();
        logConfigurator.setFileName(filePath + fileName + ".log");
        logConfigurator.setRootLevel(Level.DEBUG);
        logConfigurator.setUseFileAppender(flag);
        logConfigurator.setLevel("org.apache", Level.ERROR);
        logConfigurator.setFilePattern("%d{yyyy-MM-dd HH:mm:ss.SS} " + +Process.myPid() + "-%t/%p/%m%n");// 设置日志文件格式
        logConfigurator.setMaxFileSize(MAX_SIZE);// 日志文件大小
        logConfigurator.setMaxBackupSize(10);// 最多生成日志文件数量
        logConfigurator.setImmediateFlush(true);
        logConfigurator.configure();
    }
}

```


最后在使用日志输出的时候使用自定义的Log替换android自带的Log类引用即可。