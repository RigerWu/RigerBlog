---
title: SpringBoot Logback日志打印详解
date: 2021-01-13 22:12:40
index_img: https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210113155958249.png
categories:
  - spring boot
tags:
  - log
  - logback
---

# SpringBoot Logback 日志打印详解

## 1. 为什么不要用 println

日志打印对于后端服务来说是极其重要的，很多时候我们定位问题都需要去 log 里找

Java 初学阶段，我们会经常用 `System.out.println()`来打印 log，但是实际开发千万不要这么做

一来不够灵活，不能存文件、不能区分级别、不能配置开关

二来`println`有很大的性能问题

我们看看 `println`的源码：

```java
public void println(String x) {
  synchronized (this) {
    print(x);
    newLine();
  }
}
// print() 后面会调用 write()
private void write(String s) {
  try {
    synchronized (this) {
      ensureOpen();
      textOut.write(s);
      textOut.flushBuffer();
      charOut.flushBuffer();
      if (autoFlush && (s.indexOf('\n') >= 0))
        out.flush();
    }
  }
  catch (InterruptedIOException x) {
    Thread.currentThread().interrupt();
  }
  catch (IOException x) {
    trouble = true;
  }
}
private void newLine() {
  try {
    synchronized (this) {
      ensureOpen();
      textOut.newLine();
      textOut.flushBuffer();
      charOut.flushBuffer();
      if (autoFlush)
        out.flush();
    }
  }
  catch (InterruptedIOException x) {
    Thread.currentThread().interrupt();
  }
  catch (IOException x) {
    trouble = true;
  }
}
```

可以看到，`System.out.println`方法使用了许多 `synchronized` 关键字，如果打印的内容比较长，并发高的情况下十分影响性能

所以开发中，我们都会使用相应的日志框架来打印日志，可以用配置文件实现灵活的配置

## 2. 日志门面与日志实现

日志框架分为两类，门面和实现，门面可以理解为定义了日志要怎么打，实现就是具体去打日志

门面：JCL(Jakarta Commons Logging) jboss-logging **SLF4j**

实现：Log4j JUL(java.util.logging) Log4j2 **Logback**

SpringBoot 使用 SLF4j 作为日志门面

如果我们使用 SpringBoot 的相应 starters，那么默认使用的日志实现框架就是 [Logback](http://logback.qos.ch/index.html)

![image-20210113113141250](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210113113141250.png)

各个日志实现的配置文件：

|  框架   |                                 配置文件                                  |
| :-----: | :-----------------------------------------------------------------------: |
| Logback | logback-spring.xml, logback-spring.groovy, logback.xml, or logback.groovy |
| Log4j2  |                      log4j2-spring.xml or log4j2.xml                      |
|   JUL   |                            logging.properties                             |

Spring Boot 官方推荐使用 `-spring`后缀的配置文件，这样 Spring 可以完全控制日志的初始化，日志也可以使用 Profile

## 3. Logback 介绍

Logback 是以前非常流畅的日志框架 Log4j 的继任者，它更小，但性能更强，更多优点看[官方文档](http://logback.qos.ch/reasonsToSwitch.html)

它实现了日志门面 Slf4j，我们日志打印使用接口抽象即可：

```java
private final Log log = LogFactory.getLog(ClassName.class);
```

这样允许我们方便地切换到别的日志框架如 Log4j

通常我们可以引用 Lombok，这样直接在类上加上 `@Slf4j`注解就可以直接打印 log：

```xml
	<dependency>
		<groupId>org.projectlombok</groupId>
		<artifactId>lombok</artifactId>
		<version>1.18.16</version>
		<scope>provided</scope>
	</dependency>
```

```java
@RestController
@Slf4j
public class LogController {

    @GetMapping("/log")
    public String hello() {
        String s = "log print test";
        log.debug(s);
        log.info(s);
        log.error(s);
        log.warn(s);
        return s;
    }
}
```

## 4. Logback 配置文件

这里给出我常用的`logback-spring.xml`，注释很详细，可以自行修改内容

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL，如果设置为WARN，则低于WARN的信息都不会输出 -->
<!-- scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true -->
<!-- scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。 -->
<!-- debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。 -->
<configuration scan="true" scanPeriod="60 seconds">
    <!-- ERROR日志格式,后面在对应的appender定义中指定pattern为此值，即可以按照此处的日志格式进行输出 -->
    <property name="FILE_ERROR_PATTERN"
              value="${FILE_LOG_PATTERN:-%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}} ${LOG_LEVEL_PATTERN:-%5p} ${PID:- } --- [%t] %-40.40logger{39} %file:%line: %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>

    <!--  日志文件位置  -->
    <property name="LOG_HOME" value="logs"/>
    <property name="LOG_FILE_SIZE" value="100MB"/>

    <!-- 彩色日志 -->
    <!-- 彩色日志依赖的渲染类 -->
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter"/>
    <conversionRule conversionWord="wex"
                    converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter"/>
    <conversionRule conversionWord="wEx"
                    converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter"/>


    <!-- 彩色日志格式,后面在对应的appender定义中指定pattern为此值，即可以按照此处的日志格式进行输出  -->
    <!-- 默认情况下， 自带的CONSOLE_LOG_PATTERN属性，即为彩色的，此处重新定义了个新的属性，用于演示彩色控制台的使用示例-->
    <property name="CONSOLE_LOG_PATTERN_COLOR"
              value="${CONSOLE_LOG_PATTERN:-%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{0}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>


    <!-- 解说下具体的pattern中属性的含义,详细参见http://logback.qos.ch/manual/layouts.html-->
    <!-- %logger{x}，或者写为%c{x}，表示类名信息，如果x为0则表示仅输出类名，如果x大于0则尝试输出全路径，并按照指定的x值进行缩写。
        比如com.test.sample.Main：
        设置%logger{0}为Main
        设置%logger{5}可缩写为c.t.s.Main
        设置%logger{20}为com.test.sample.Main-->
    <!--  %d表示日期信息(或者%date)，默认格式为【yyyy-MM-dd HH:mm:ss,SSS】，可以通过%d{x}自定义格式，比如%d{yyyy-MM-dd HH:mm:ss.SSS}-->
    <!--  %p表示日志的level,或者%le\%level-->
    <!--  %L表示相对应的行号，或者%line-->
    <!--  %m表示实际的需要输出的日志内容字符串,或者%msg\%message都可以-->
    <!--  %replace(p){r,t}表示对p中给定的内容进行字符串替换，将r替换为t，支持正则，比如将p中的换行替换为下划线，防止日志攻击-->
    <!--  %M表示方法名称，或者%method也可以-->
    <!--  %t表示线程名称，或者%thread也可以-->
    <!-- %ex{x}用于指定错误异常堆栈的打印策略，x可以为short\full或者具体数字，表示打印多少行堆栈-->
    <!--  %n表示换行-->
    <!-- 字符串对齐、截取、格式化说明：
         在上述的各个占位符中，在%和具体字符之间，可以插入格式化指令，以%c为例，如下：
         %20c 表示%c的内容如果不足20位，则在左侧以空格填充满20长度
         %-20c 与%20c相似，区别在于，会在右侧以空格填满20长度
         %.20c 表示%c内容如果超过20，则会截取掉开头的内容，只留下右侧20位长度
         %.-20c 与%.20c相似，区别在于，会截取掉末尾的内容，只留下开头20位长度
         以此类推，还可以组合出如下使用方式：
         %20.30c 表示最短20位，最长30位，如果不足20位则左侧补齐空格，如果超过30位则丢弃左侧开头的字符串
         %-20.30c 和%20.30c类似，区别在于不足20位的时候，在右侧补齐空格
         %-20.-30c 和%-20.30c类似，区别在于超过30位的时候会丢弃结尾部分的字符串
         %20.-30c 和20.30c类似，区别在于超过30位的时候会丢弃结尾部分的字符串
        -->

    <property name="SELF_DEFINE_LOG_PATTERN"
              value="[%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint}] [%clr(%5.5p){magenta}] [%clr(%5.10t){faint}] [%clr(%30.30c{0}.%20.20M:%4.4L){cyan}]  -> %replace(%.-300m){'\r\n','__'}%ex{full}%n"/>

    <!-- Spring Boot 提供了一个默认的 xml 配置，可以按照如下方式引入 -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <!-- 引入application.yml配置文件，后续可以直接引用里面的属性值-->
    <!-- 注意：在此文件中直接${}的方式引用application.yml里面配置可能会报错.
    因为logback.xml的加载顺序早于springboot的application.yml (或application.properties) 配置文件。
    所以可以先springProperty的方式定义个本地变量引入进来，再进行引用此本地变量-->
    <!-- 可以将一些公共的内容放到application.yml里面去配置，然后此文件中引用，后续可以避免修改此xml，简单的参数直接修改下application.yml就行了-->
    <!-- 比如将日志存放路径与文件大小信息从配置中读取，这样dev和prod可以指定不同的逻辑-->
    <!--    <property resource="application.yml"/>-->
    <!--    <springProperty scope="context" name="LOG_HOME" source="selfdefine.logfile.rootPath"/>-->
    <!--    <springProperty scope="context" name="LOG_FILE_SIZE" source="selfdefine.logfile.max-size"/>-->

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
        </filter>
        <!-- 指定此append对应的日志内容的格式-->
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN_COLOR}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!--  生产慎用!!! 打印方法名和行号会对性能有损耗,服务无性能要求可用  -->
    <appender name="CONSOLE2" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
        </filter>

        <!-- 指定此append对应的日志内容的格式-->
        <encoder>
            <pattern>${SELF_DEFINE_LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>

    </appender>

    <!--  文件输出，默认Info级别  -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--如果只是想要 Error 级别的日志，那么需要过滤一下，默认是 info 级别的，ThresholdFilter-->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <!-- 指定过滤的日志级别，只有等于或者高于此级别的，才会通过此appender进行输出-->
            <level>Info</level>
        </filter>
        <!--日志名称，如果没有File 属性，那么只会使用FileNamePattern的文件路径规则如果同时有<File>和<FileNamePattern>，那么
        当天日志是<File>，明天会自动把今天的日志改名为今天的日期。即，<File> 的日志都是当天的。-->
        <File>${LOG_HOME}/web-log.log</File>
        <!--滚动策略，按照时间滚动 TimeBasedRollingPolicy-->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--文件路径，支持相对路径或者绝对路径（尽量避免相对路径，通过绝对路径保证存储位置的固定）,定义了日志的切分方式——把每一天的日志归档到一个文件中,以防止日志填满整个磁盘空间-->
            <!-- 指定文件的路径以及对应的文件命名格式，其中%i表示递增标识ID序号，日志切换绕接的时候递增-->
            <FileNamePattern>${LOG_HOME}/Log._%d{yyyy-MM-dd}.part_%i.log</FileNamePattern>
            <!--保留日志天数-->
            <maxHistory>180</maxHistory>
            <!--用来指定日志文件的上限大小，那么到了这个值，就会删除旧的日志-->
            <!--<totalSizeCap>1GB</totalSizeCap>-->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <!-- maxFileSize:这是活动文件的大小-->
                <maxFileSize>${LOG_FILE_SIZE}</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <!--<triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">-->
        <!--<maxFileSize>1KB</maxFileSize>-->
        <!--</triggeringPolicy>-->

        <!-- 指定此append对应的日志内容的格式-->
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
    </appender>

    <!-- 指定日志输出到哪些位置、以及root日志的输出level-->
    <!-- 默认情况下的使用，任何spring profile值情况下都会使用下面的配置，即输出到console中-->
    <root level="info">
        <appender-ref ref="CONSOLE2"/>
    </root>

    <!-- 在SpringBoot中，可以通过springProfile属性来实现在不同环境上执行不同的输出策略，如下示例中指定pro和dev上有不同的策略-->
    <!-- 指定在prod环境使用的输出配置-->
    <springProfile name="pro">
        <root level="info">
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>

    <!-- 指定在dev环境使用的输出配置-->
    <springProfile name="dev">
        <root level="info">
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>
</configuration>
```

这里还是要再强调一下，对于并发或性能要求高的服务，不要打印方法名和行号，会影响性能

参见 Logbak 的官方文档

> Generating the method name is not particularly fast. Thus, its use should be avoided unless execution speed is not an issue.
>
> Generating the line number information is not particularly fast. Thus, its use should be avoided unless execution speed is not an issue.

看一下使用我自定义的 Pattern 打印的带颜色行号的日志格式：

![image-20210113155958249](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210113155958249.png)

## 5. 日志等级

默认等级在配置文件里配置，我们还可以手动配置指定包名、类名的日志等级：

```yaml
logging:
  level:
    root: "warn"
    org.springframework.web: "debug"
    org.hibernate: "error"
```

还可以给一些包分组，分组后一次性配置：

```yaml
logging:
  group:
    tomcat: "org.apache.catalina,org.apache.coyote,org.apache.tomcat"
  level:
  tomcat: "trace"
```

Spring Boot 预定义了两个分组

| Name | Loggers                                                                                                                                                                                              |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| web  | org.springframework.core.codec, org.springframework.http,org.springframework.web, org.springframework.boot.actuate.endpoint.web, org.springframework.boot.web.servlet.ServletContextInitializerBeans |
| sql  | org.springframework.jdbc.core, org.hibernate.SQL, org.jooq.tools.LoggerListener                                                                                                                      |

## 6. 统一日志输出

我用了一个开源项目 XXX，它日志框架用的是 Log4j ，我怎么统一用 SLF4j+Logback 来输出？

SLF4j 官方文档给出了下图的指引：

![img](https://riger.oss-cn-shanghai.aliyuncs.com/img/legacy.png)

已切换 Log4j 为例：

我们只需要引入 `log4j-over-slf4j.jar`,然后把开源项目的 log4j 依赖排除即可

```xml
<!-- 这三个依赖可以根据需要选择 -->
<dependency>
   <groupId>org.slf4j</groupId>
   <artifactId>jcl-over-slf4j</artifactId>
    <version>${org.slf4j-version}</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>log4j-over-slf4j</artifactId>
    <version>${org.slf4j-version}</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jul-to-slf4j</artifactId>
    <version>${org.slf4j-version}</version>
</dependency>
```

篇幅有限，分布式环境的日志统一存储，下一篇再讲。

源码及脚本都在[Github](https://github.com/RigerWu/web-starter-demos)上

Enjoy it!
