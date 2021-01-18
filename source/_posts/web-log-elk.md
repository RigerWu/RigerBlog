---
title: 使用ELK记录微服务日志
date: 2021-01-17 22:12:40
index_img: https://riger.oss-cn-shanghai.aliyuncs.com/img/elk-0931238.png
categories:
  - spring boot
tags:
  - log
  - logstash
  - elk
---

# 使用ELK记录微服务日志

## 1. 简介

上一节我们讲解了 Logback 的配置，完成了日志打印到文件的第一步

一般来说，后台服务规模比较小的情况下，这样是没问题的，但是一旦分布式部署众多服务器，日志的查询和管理就成了很大的问题

这个时候我们可以使用比较成熟的分布式日志解决方案：ELK

ELK 是 Elasticsearch、Logstash、Kibana 的缩写

简单来说就是通过 Logstash 收集处理日志，然后存储到 Elasticsearch，在通过 Kinbana 进行可视化的搜索、查询、汇聚分析。

我们使用使用 SpringBoot 来构建微服务，可以配置 [logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder) 通过tcp或udp把产生的日志传送到 Logstash服务

简单流程如图：

## ![elk](https://riger.oss-cn-shanghai.aliyuncs.com/img/elk-0931238.png)

## 2. 搭建

为了方便起见，我们使用开源项目 [docker-elk](https://github.com/deviantony/docker-elk) 的配置来进行安装

环境要求：

- Docker 版本 17.05 以上 `docker version`
- Docker Compose 版本 1.20.0 以上 `docker-compose version`
- 2G 内存以上

这里我使用我电脑上装的一台虚拟机，ip 是 `192.168.0.2`

在服务器上 clone 项目：

```bash
git clone https://github.com/deviantony/docker-elk.git
# Redhat 和 CentOS 执行一下以下语句确保能正常使用
chcon -R system_u:object_r:admin_home_t:s0 docker-elk/
```

起目录结构如下：

```
docker-elk/
├── docker-compose.yml
├── docker-stack.yml
├── elasticsearch
│   ├── config
│   └── Dockerfile
├── .env
├── extensions
│   ├── apm-server
│   ├── curator
│   ├── enterprise-search
│   ├── logspout
│   ├── metricbeat
│   └── README.md
├── kibana
│   ├── config
│   └── Dockerfile
├── LICENSE
├── logstash
│   ├── config
│   ├── Dockerfile
│   └── pipeline
└── README.md
```

`.env` 文件定义了使用的 Elasticsearch 版本，我这里用是 `7.10.1`，有需要可以自行修改

然后我们修改一下 Logstash 的 build 配置，安装 json_lines 插件

```bash
vi logstash/Dockerfile
# 添加下面一行在最后
RUN bin/logstash-plugin install logstash-codec-json_lines
```

再修改 Logstash 的配置文件 `vi logstash/pipeline/logstash.conf`，配置如下：

```
input {
        beats {
                port => 5044
        }

        tcp {
                port => 5000
                hots => "0.0.0.0"
                codec => json_lines
        }
}

## Add your filters / logstash plugins configuration here

output {
        elasticsearch {
                hosts => "elasticsearch:9200"
                user => "elastic"
                password => "changeme"
                ecs_compatibility => disabled
                codec => json
        }
}
```

我们可以直接启动：

```bash
docker-compose up
```

![image-20210117203759711](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210117203759711.png)

第一次安装会下载镜像，并在镜像基础上 build，需要耐心等待

![image-20210117211638847](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210117211638847.png)

当我们看到三个 done，并且容器也开始打印 log 的时候，就启动完成了，后面 还需要等待 1 分钟左右，Kibana 才可以访问

我们访问之前，先用命令访问 Kibana Api，创建一个 index pattern：

```bash
curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 7.10.2' \
    -u elastic:changeme \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'

# 可以看到成功创建了
[root@localhost docker-elk]# curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
>     -H 'Content-Type: application/json' \
>     -H 'kbn-version: 7.10.2' \
>     -u elastic:changeme \
>     -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
HTTP/1.1 200 OK
kbn-name: kibana
kbn-license-sig: 9c18aa7a665c803facb3814b41effaf3fd82a9af5581d75bf4d30992a38908a7
content-type: application/json; charset=utf-8
cache-control: private, no-cache, no-store, must-revalidate
content-length: 280
Date: Sun, 17 Jan 2021 13:26:40 GMT
Connection: keep-alive

{"type":"index-pattern","id":"a1a4cb40-58c7-11eb-a33e-6bbb0a638f52","attributes":{"title":"logstash-*","timeFieldName":"@timestamp"},"references":[],"migrationVersion":{"index-pattern":"7.6.0"},"updated_at":"2021-01-17T13:26:39.859Z","version":"Wzc2LDNd","namespaces":["default"]}
```

## 3. Kibana配置

直接访问 192.168.0.2:5601  

![image-20210117230703734](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210117230703734.png)

输入 docker-elk 配置的默认账户名：`elastic` 密码：`changeme`即可访问

我这里测试使用就不改密码了 ，生产环境建议按如下步骤修改密码：

```
# 1. 批量生产随机密码,记录下密码
docker-compose exec -T elasticsearch bin/elasticsearch-setup-passwords auto --batch
# 2. 删除 docker-compose.yml 中的 ELASTIC_PASSWORD
# 3. 修改kibana/config/kibana.yml 使用 kibana_system 用户和你自己的密码(小于7.8.0版本可以用 kibana y用户)
#    修改logstash/config/logstash.yml 使用 logstash_system 用户和你自己的密码
#    修改logstash/pipeline/logstash.conf 修改 elastic 用户的密码
```

我们点击左侧菜单栏中的 Disocver：

![image-20210118090532933](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210118090532933.png)

可以看到，我们之前创建的 index pattern 已经有了：

![image-20210118090748210](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210118090748210.png)

不过这个时候还没有数据，我们接着往下看

## 4. 微服务配置

这里还是再我们上回的 web-log 的基础上修改

引入 logstash-logback-encoder：

```xml
<!--集成logstash-->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>6.6</version>
</dependency>
```

为了不影响之前的配置，我们拷贝一份 `logback-spring-logstash.xml`，并增加一个 dev profile：

![image-20210118093533712](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210118093533712.png)

dev 的 yml 配置如下，这里我们指定配置文件，和 Logstash的 host：

```yaml
server:
  port: 8080
spring:
  application:
    name: web-simple
logstash:
  host: 192.168.0.2
logging:
  level:
    root: debug
  config: classpath:logback-spring-logstash.xml
```

log 完整配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL，如果设置为WARN，则低于WARN的信息都不会输出 -->
<!-- scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true -->
<!-- scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。 -->
<!-- debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。 -->
<configuration scan="true" scanPeriod="60 seconds">

    <!--LogStash访问host-->
    <springProperty name="LOG_STASH_HOST" scope="context" source="logstash.host" defaultValue="localhost"/>

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

    <!--日志输出到LogStash-->
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>${LOG_STASH_HOST}:5000</destination>
        <encoder charset="UTF-8" class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>Asia/Shanghai</timeZone>
                </timestamp>
                <!--自定义日志输出格式-->
                <pattern>
                    <pattern>
                        {
                        "project": "web-mall",
                        "level": "%level",
                        "service": "${APP_NAME:-}",
                        "pid": "${PID:-}",
                        "thread": "%thread",
                        "class": "%logger",
                        "message": "%message",
                        "stack_trace": "%exception{20}"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
        <!--当有多个LogStash服务时，设置访问策略为轮询-->
        <connectionStrategy>
            <roundRobin>
                <connectionTTL>5 minutes</connectionTTL>
            </roundRobin>
        </connectionStrategy>
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
            <appender-ref ref="LOGSTASH"/>
        </root>
    </springProfile>
</configuration>
```

然后在 IDEA 的启动选项里指定 dev ，启动应用：

![image-20210118094236012](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210118094236012.png)

再访问一下测试请求：

![image-20210118094448871](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210118094448871.png)

## 5. 日志查询

我们刷新 Kibana：

![image-20210118094657559](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210118094657559.png)

由于我们日志等级是 debug，所以进来了大量日志

这个时候左侧可以看到 左侧很多字段都是 unknown field，我们的日志内容在 `message`里，我们需要刷新一下字段：

![image-20210118095848233](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210118095848233.png)

再回来刷新就发现字段已经正常了：

![image-20210118101245092](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210118101245092.png)

我前面的日志打印了时间戳：`1610934226338`

我们搜索一下看看，搜索栏输入：`message : *1610934226338*`

![image-20210118101345054](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210118101345054.png)

可以看到，能正常搜索到日志

我们展开日志，可以更清晰地看到信息：

![image-20210118101404856](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210118101404856.png)

我们还可以按我们想要的格式展现日志：比如 project + ip + message，在左侧上点 + 号添加：

![image-20210118101750772](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210118101750772.png)

至此，我们完成了日志的统一存储、查询

其实我们还可以分场景收集日志，比如分访问日志、应用日志、错误日志，可以用于访问量统计，日志查询，错误监控等

在 LogStash 的 input 配置多个端口，配置不同的 type，然后根据 type 创建不同的 ES index 即可

可以根据业务需要去配置，这里就不细展开了

好了，本篇到此结束，希望对你有帮助！

源码及脚本都在[Github](https://github.com/RigerWu/web-starter-demos)上

Enjoy it!



