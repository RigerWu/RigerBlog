---
title: Spring Boot Admin 应用监控
date: 2021-01-28 22:12:40
index_img: https://riger.oss-cn-shanghai.aliyuncs.com/img/20210128161315.png
categories:
  - spring boot
tags:
  - admin
  - tool
---

# Spring Boot Admin 应用监控

## 1. 简介

[Spring Boot Admin](https://github.com/codecentric/spring-boot-admin) 可以帮助我们非常方便得查看应用的：

- 健康信息
- 内存指标
- 日志信息
- JVM 系统变量和环境变量
- 线程和堆栈信息
- 等等

还能进行状态通知用于告警，集成非常简单，简单又好用。

## 2. 集成

Admin 需要单独部署一个微服务，作为 Server 端，并提供 UI 的后台服务，然后我们的应用接入Server即可。

### Server

新建一个 `admin-server`, pom 引入 amdin 依赖，最新版本在[这里](https://github.com/codecentric/spring-boot-admin/releases)查：

```xml
<!--    admin server    -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
    <version>${spring.boot.admin.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

yml 配置：

```yaml
server:
  port: 8000
spring:
  security:
    user:
      name: riger
      password: riger
```

Security 配置，主要是为了增加密码登录：

```java
package com.rigerwu.web.admin.server.config;

import de.codecentric.boot.admin.server.config.AdminServerProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.web.authentication.SavedRequestAwareAuthenticationSuccessHandler;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

/**
 * created by riger on 2021/1/28
 */
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    private final String adminContextPath;

    public WebSecurityConfig(AdminServerProperties adminServerProperties) {
        this.adminContextPath = adminServerProperties.getContextPath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("redirectTo");
        successHandler.setDefaultTargetUrl(adminContextPath + "/");

        http.authorizeRequests()
                //1.配置所有静态资源和登录页可以公开访问
                .antMatchers(adminContextPath + "/assets/**").permitAll()
                .antMatchers(adminContextPath + "/login").permitAll()
                .anyRequest().authenticated()
                .and()
                //2.配置登录和登出路径
                .formLogin().loginPage(adminContextPath + "/login").successHandler(successHandler).and()
                .logout().logoutUrl(adminContextPath + "/logout").and()
                //3.开启http basic支持，admin-client注册时需要使用
                .httpBasic().and()
                .csrf()
                //4.开启基于cookie的csrf保护
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                //5.忽略这些路径的csrf保护以便admin-client注册
                .ignoringAntMatchers(
                        adminContextPath + "/instances",
                        adminContextPath + "/actuator/**"
                );
    }
}
```

最后在 Application 上开启：`@EnableAdminServer`

```java
@SpringBootApplication
@EnableAdminServer
public class AdminServerApplication {
    public static void main(String[] args){
        SpringApplication.run(AdminServerApplication.class, args);
    }
}
```

启动Server，然后访问：http://localhost:8000

<img src="https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210128154955409.png" alt="image-20210128154955409" style="zoom:67%;" />



### Client

引入依赖：

```xml
<!--    admin client    -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
    <version>${spring.boot.admin.version}</version>
</dependency>
```

配置文件，由于我们 Server 配置了账号密码，所以客户端也要配置，才能接入 Server：

```yaml
server:
  port: 8080
spring:
  application:
    name: admin-client
  boot:
    admin:
      client:
        url: http://localhost:8000
        username: riger
        password: riger

management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: always

logging:
  file:
    name: client.log # 要配置log文件名,才能在Admin里查看log
```

再启动客户端

## 3. 使用

我们使用预设的账号密码：riger/riger 登录：

![image-20210128155041985](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210128155041985.png)

可以看到我们的应用已经注册在 Admin 中，点击可以查看详细信息：

![image-20210128155822906](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210128155822906.png)

![image-20210128155414534](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210128155414534.png)

![image-20210128155716702](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210128155716702.png)

![image-20210128155456504](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210128155456504.png)

还能直接查看到应用 log，并可以下载 log：

![image-20210128155549594](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210128155549594.png)

非常强大且方便

## 4. 邮件告警

这里简单介绍一下邮件告警的配置使用, Server 引入：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

增加邮件配置，可配置多个通知邮箱，用 `,`隔开：

```yaml
server:
  port: 8000
spring:
  security:
    user:
      name: riger
      password: riger
  mail:
    host: smtp.qiye.163.com
    port: 465
    username: yourmail@163.com
    password: password
    protocol: smtp
    test-connection: true
    default-encoding: UTF-8
    properties:
      mail.smtp.auth: true
      mail.smtp.ssl.enable: true

  boot:
    admin:
      notify:
        mail:
          # from 163邮箱必须要配置，否则报503，别的邮箱没测
          from: yourmail@163.com
          to: mail1@163.com,mail2@163.com
```

我们重启 Server，然后关掉我们的 Client 后，收到了邮件：

![image-20210128160802389](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210128160802389.png)

好了，Spring Boot Admin 就介绍到这，希望有所帮助。

源码及脚本都在[Github](https://github.com/RigerWu/web-starter-demos)上

Enjoy it!

### 参考资料

- [Spring Boot Admin Document](https://codecentric.github.io/spring-boot-admin/2.3.1/#getting-started)