---
title: SpringBoot工程的创建运行发布
date: 2021-01-10 13:09:12
index_img: https://riger.oss-cn-shanghai.aliyuncs.com/img/20210112150409.png
categories: 
- spring boot
tags: 
- spring boot
- idea
---

# Idea创建SrpingBoot工程

## 1. 创建

由于博客的示例代码都放在`web-starter-demos`项目里，所以这里我创建的是`module`而不是`project`

我们使用官方的`Spring Initializr`来初始化工程，jdk选择1.8：

![image-20210112102036076](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112102036076.png)

填好`Group`,`Artifact`,选择`Java Version`为8，点击`Next`：

![image-20210112102749459](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112102749459.png)

`Spring Boot`我选择目前最新的稳定版本2.4.1，你可以在[官网](https://spring.io/projects/spring-boot#learn)查询最新GA版本

同时这个界面还能让我们方便得选择一些开发常用的依赖，本文的简单项目，我们只添加以下两个，点击`Next`：

![image-20210112135337955](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112135337955.png)

填好`Module Name`，点击`Finish`：

![image-20210112103919173](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112103919173.png)

我们得到了如下目录结构的工程：

![image-20210112105357563](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112105357563.png)

## 2. 运行

我们直接点击主类的启动按钮：

![image-20210112105444193](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112105444193.png)

项目就运行起来了：

![image-20210112105538899](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112105538899.png)

可以看到默认情况下，项目运行在8080端口

## 3. 优化

作为开发者，我们电脑上肯定有装 `Maven`并配置了环境变量，所以我们不需要工程初始化带的`Maven Wrapper`

由于我这里是`module` 外层`.gitignore`已经做了全局配置，这里也不需要，帮助文档也删掉

```shell
 rm -rf .mvn mvnw* .gitignore HELP.md
```

建议使用`yml`配置，我们把`application.properties` 改成 `application.yml`

一般建议显示指定端口号和服务名：

```yaml
server:
  port: 8080
spring:
  application:
    name: web-simple
```

编辑一下 IDE 里的运行配置：

![image-20210112135821537](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112135821537.png)

这里可以设置虚拟机参数，环境变量等：

![image-20210112140241936](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112140241936.png)

如图，一般开发的时候，我会指定比较小的堆内存大小，这样可以启动多个服务而不卡

还可以指定服务端口，比如要跑多个服务，直接点左上角复制一份配置，改个端口号就可以

以及可以指定 `profile`，比如区分开发环境、生产环境等，不需要改配置文件，即可指定启动

## 4. 请求

我们编写一个简单的`Controller`

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() {
        return "Hello World!";
    }
}
```

运行后可以直接访问，这里我使用插件[RestfullTool](https://plugins.jetbrains.com/plugin/14280-restfultool)来访问：

![image-20210112114058484](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112114058484.png)

## 5. 发布

我们在项目根目录下使用maven命令来打包项目（使用Idea的Maven插件点击按钮也可以）：

`-Dmaven.test.skip=true`可以跳过测试

```shell
  mvn clean package -Dmaven.test.skip=true
```

![image-20210112115142859](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112115142859.png)

打包完成后，`target`路径下就有了我们需要的可执行 `jar`：

![image-20210112115232952](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210112115232952.png)

然后就可以把 `jar`丢到服务端，运行起来：

```shell
java -jar web-simple-0.0.1-SNAPSHOT.jar
```

但是这样会占用控制台，可以使用 `nohup`命令在后台启动

```shell
nohup java -jar web-simple-0.0.1-SNAPSHOT.jar >/dev/null 2>&1 &
```

一般在服务器端会写一个启动脚本来控制应用的启动、关闭等操作，并且还可以配置想要的启动参数、环境变量等

这会在后续的博文中详细介绍，这里仅仅给出一个我使用的启动脚本文件：

`start.sh`

```bash
#!/bin/bash
JDK_HOME="java"
VM_OPTS="-Xms1g -Xmx1g"
SPB_OPTS="--spring.profiles.active=test"
APP_LOCATION="web-simple-0.0.1-SNAPSHOT.jar"
APP_NAME="web-simple-0.0.1-SNAPSHOT.jar"
PID_CMD="ps -ef |grep $APP_NAME |grep -v grep |awk '{print \$2}'"

start() {
  echo "=============================start=============================="
  PID=$(eval "$PID_CMD")
  if [[ -n $PID ]]; then
    echo "$APP_NAME is already running,PID is $PID"
  else
    nohup $JDK_HOME $VM_OPTS -jar $APP_LOCATION $SPB_OPTS >/dev/null 2>&1 &
    echo "nohup $JDK_HOME $VM_OPTS -jar $APP_LOCATION $SPB_OPTS >/dev/null 2>&1 &"
    PID=$(eval "$PID_CMD")
    if [[ -n $PID ]]; then
      echo "Start $APP_NAME successfully,PID is $PID"
    else
      echo "Failed to start $APP_NAME !!!"
    fi
  fi
  echo "=============================start=============================="
}

stop() {
  echo "=============================stop=============================="
  PID=$(eval "$PID_CMD")
  if [[ -n $PID ]]; then
    kill -15 "$PID"
    sleep 5
    PID=$(eval "$PID_CMD")
    if [[ -n $PID ]]; then
      echo "Stop $APP_NAME failed by kill -15 $PID,begin to kill -9 $PID"
      kill -9 "$PID"
      sleep 2
      echo "Stop $APP_NAME successfully by kill -9 $PID"
    else
      echo "Stop $APP_NAME successfully by kill -15 $PID"
    fi
  else
    echo "$APP_NAME is not running!!!"
  fi
  echo "=============================stop=============================="
}

restart() {
  echo "=============================restart=============================="
  stop
  start
  echo "=============================restart=============================="
}

status() {
  echo "=============================status=============================="
  PID=$(eval "$PID_CMD")
  if [[ -n $PID ]]; then
    echo "$APP_NAME is running,PID is $PID"
  else
    echo "$APP_NAME is not running!!!"
  fi
  echo "=============================status=============================="
}

info() {
  echo "=============================info=============================="
  echo "APP_LOCATION: $APP_LOCATION"
  echo "APP_NAME: $APP_NAME"
  echo "JDK_HOME: $JDK_HOME"
  echo "VM_OPTS: $VM_OPTS"
  echo "SPB_OPTS: $SPB_OPTS"
  echo "=============================info=============================="
}

help() {
  echo "start: start server"
  echo "stop: shutdown server"
  echo "restart: restart server"
  echo "status: display status of server"
  echo "info: display info of server"
  echo "help: help info"
}

case $1 in
start)
  start
  ;;
stop)
  stop
  ;;
restart)
  restart
  ;;
status)
  status
  ;;
info)
  info
  ;;
help)
  help
  ;;
*)
  help
  ;;
esac
exit $?
```

可以很方便地使用一下命令：

```
~ ./start.sh
start: start server
stop: shutdown server
restart: restart server
status: display status of server
info: display info of server
help: help info
~ ./start.sh start
=============================start==============================
nohup java -Xms1g -Xmx1g -jar target/web-simple-0.0.1-SNAPSHOT.jar --spring.profiles.active=dev >/dev/null 2>&1 &
Start web-simple-0.0.1-SNAPSHOT.jar successfully,PID is 6508
=============================start==============================
 ~ ./start.sh status
=============================status==============================
web-simple-0.0.1-SNAPSHOT.jar is running,PID is 6508
=============================status==============================
 ~ ./start.sh stop  
=============================stop==============================
Stop web-simple-0.0.1-SNAPSHOT.jar successfully by kill -15 
=============================stop==============================
 ~ 
```

源码及脚本都在[Github](https://github.com/RigerWu/web-starter-demos)上

Enjoy it!