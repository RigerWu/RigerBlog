---
title: Mybatis基本使用入门
date: 2021-01-23 22:10:40
index_img: https://riger.oss-cn-shanghai.aliyuncs.com/img/mybatis-superbird-small.png
categories:
  - spring boot
tags:
  - mybatis
  - db
---

# Mybatis基本使用入门

## 1. Mybatis 简介

[Mybatis](https://mybatis.org/mybatis-3/zh/index.html) 是一款优秀的持久层框架，它在简化了我们对数据库的操作和资源管理的同时，允许开发者自定义 SQL，保留了灵活性，是目前国内互联网环境中用得最多的 ORM 框架

它支持使用 XML 和 注解 两种方式来配置，一般来说我们推荐用 XML 配置：

一来可以集中配置，把配置放在一起；二来有比较好的插件支持，书写、生成都很方便，代码提示也更好

## 2. 环境准备

我们使用 docker 在本地搭建一个的 Mysql 环境：

```
# 拉取镜像
docker pull mysql:5.7
# 这里重点是演示 Mybatis 就不映射配置文件和数据文件了
docker run --name mysql \
-p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 \
-d mysql:5.7
```

我们使用 Navicat 等工具连接数据库，创建一个数据库：`mybatis`

再准备一点数据，用作后面的测试：

```sql
DROP TABLE IF EXISTS user;
CREATE TABLE user
(
	id BIGINT(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
	name VARCHAR(30) NULL DEFAULT NULL COMMENT '姓名',
	age INT(11) NULL DEFAULT NULL COMMENT '年龄',
	email VARCHAR(50) NULL DEFAULT NULL COMMENT '邮箱',
	PRIMARY KEY (id)
);
DELETE FROM user;
INSERT INTO user (id, name, age, email) VALUES
(1, 'Tom', 18, 'tom@rigerwu.com'),
(2, 'Lily', 20, 'lily@rigerwu.com'),
(3, 'Jack', 28, 'jack@rigerwu.com'),
(4, 'Rose', 21, 'rose@rigerwu.com'),
(5, 'Bob', 24, 'bob@rigerwu.com');
```

![image-20210122223730914](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210122223730914.png)
接下来我们直接在 IDEA 里配置数据库（为了后面的代码生成和提示）：

<img src="https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210122223103159.png" alt="image-20210122223103159" />

我们输入相应的信息，测试通过后点击 OK：

![image-20210122224100569](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210122224100569.png)

这样我们就可以直接查看到表和数据了：

![image-20210122224207567](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210122224207567.png)

双击表名可以直接查询数据：

![image-20210122224244318](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210122224244318.png)

## 3. 依赖引入与配置

新建一个 `web-mybatis` 项目，引入 mybatis 的 starter，和 druid 数据源：

你们可以再 Maven 仓库查找 [mybatis-starter](https://mvnrepository.com/artifact/org.mybatis.spring.boot/mybatis-spring-boot-starter) 和 [druid](https://mvnrepository.com/artifact/com.alibaba/druid-spring-boot-starter) 的最新版本

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.4</version>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.4</version>
</dependency>
```

yml 里做相应配置：

```yaml
server:
  port: 8090
spring:
  application:
    name: web-mybatis
  datasource:
    username: root
    password: 123456
    url: jdbc:mysql://localhost:3306/mybatis?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
    #   数据源其他配置
    druid:
      initial-size: 10 # 初始化时建立物理连接的个数。初始化发生在显示调用init方法，或者第一次getConnection时
      min-idle: 10 # 最小连接池数量
      maxActive: 200 # 最大连接池数量
      maxWait: 60000 # 获取连接时最大等待时间，单位毫秒。配置了maxWait之后，缺省启用公平锁，并发效率会有所下降，如果需要可以通过配置
      timeBetweenEvictionRunsMillis: 60000 # 关闭空闲连接的检测时间间隔.Destroy线程会检测连接的间隔时间，如果连接空闲时间大于等于minEvictableIdleTimeMillis则关闭物理连接。
      minEvictableIdleTimeMillis: 300000 # 连接的最小生存时间.连接保持空闲而不被驱逐的最小时间
      validationQuery: SELECT 1 # 验证数据库服务可用性的sql.用来检测连接是否有效的sql 因数据库方言而差, 例如 oracle 应该写成 SELECT 'x' FROM DUAL
      testWhileIdle: true # 申请连接时检测空闲时间，根据空闲时间再检测连接是否有效.建议配置为true，不影响性能，并且保证安全性。申请连接的时候检测，如果空闲时间大于timeBetweenEvictionRun
      testOnBorrow: false # 申请连接时直接检测连接是否有效.申请连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。
      testOnReturn: false # 归还连接时检测连接是否有效.归还连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。
      poolPreparedStatements: true # 开启PSCache
      maxPoolPreparedStatementPerConnectionSize: 20 #设置PSCache值
      connectionErrorRetryAttempts: 3 # 连接出错后再尝试连接三次
      breakAfterAcquireFailure: true # 数据库服务宕机自动重连机制
      timeBetweenConnectErrorMillis: 300000 # 连接出错后重试时间间隔
      asyncInit: true # 异步初始化策略
      remove-abandoned: true # 是否自动回收超时连接
      remove-abandoned-timeout: 1800 # 超时时间(以秒数为单位)
      transaction-query-timeout: 6000 # 事务超时时间
      filters: stat,wall,slf4j
      connectionProperties: druid.stat.mergeSql\=true;druid.stat.slowSqlMillis\=5000
      web-stat-filter:
        enabled: true
        url-pattern: /*
        exclusions: "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*"
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
        allow:
        #        deny:
        reset-enable: true
        login-username: riger
        login-password: riger
      filter:
        slf4j:
          enabled: true
          statement-executable-sql-log-enable: true

mybatis:
  # mapper 映射文件位置
  mapper-locations: classpath:mapper/*.xml
  configuration:
    # 开启映射驼峰和下划线
    map-underscore-to-camel-case: true
  type-aliases-package: com.rigerwu.web.mybatis.entity
# 让日志打印Sql
logging:
  level:
    com.rigerwu.web.mybatis.mapper: debug
```

启动类上加上 MapperScan，这样我们不用每个接口上都标上 `@Mapper` 注解了：

```java
@MapperScan(basePackages = {"com.rigerwu.web.mybatis.mapper"})
```



## 4. 插件的基本使用

接下来，重点来了！我们打开 Settings -> plugins -> marketplace -> 搜索 `MybatisX`，下载安装：

<img src="https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210122223103159.png" alt="image-20210122223103159" style="zoom:70%;" />

今后我们怎么写 XML 和 接口？不用写，直接生成！我们在 User 表上右键，选择 Mybatis-Generator ：

<img src="https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210122224558967.png" alt="image-20210122224558967" style="zoom: 80%;" />

填好包名和路径，点击OK，这里为了代码简洁，我还勾选了 Lombok：

![image-20210123222114081](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210123222114081.png)

点击后生成了三个文件（红色未 add 的三个，这里我方便演示没再分包了）：

![image-20210123222142905](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210123222142905.png)

User.java

```java
package com.rigerwu.web.mybatis;

import java.io.Serializable;
import lombok.Data;

/**
 * @TableName user
 */
@Data
public class User implements Serializable {
    /**
     * 主键ID
     *
     * @mbg.generated Fri Jan 22 22:51:56 CST 2021
     */
    private Long id;

    /**
     * 姓名
     *
     * @mbg.generated Fri Jan 22 22:51:56 CST 2021
     */
    private String name;

    /**
     * 年龄
     *
     * @mbg.generated Fri Jan 22 22:51:56 CST 2021
     */
    private Integer age;

    /**
     * 邮箱
     *
     * @mbg.generated Fri Jan 22 22:51:56 CST 2021
     */
    private String email;

    /**
     * This field was generated by MyBatis Generator.
     * This field corresponds to the database table user
     *
     * @mbg.generated Fri Jan 22 22:51:56 CST 2021
     */
    private static final long serialVersionUID = 1L;
}
```

UserDao.java

```java
package com.rigerwu.web.mybatis;

import com.rigerwu.web.mybatis.User;

/**
 * @Entity com.rigerwu.web.mybatis.User
 */
public interface UserDao {
    /**
     * @mbg.generated
     */
    int deleteByPrimaryKey(Long id);

    /**
     * @mbg.generated
     */
    int insert(User record);

    /**
     * @mbg.generated
     */
    int insertSelective(User record);

    /**
     * @mbg.generated
     */
    User selectByPrimaryKey(Long id);

    /**
     * @mbg.generated
     */
    int updateByPrimaryKeySelective(User record);

    /**
     * @mbg.generated
     */
    int updateByPrimaryKey(User record);
}
```

UserDao.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.rigerwu.web.mybatis.UserDao">
  <resultMap id="BaseResultMap" type="com.rigerwu.web.mybatis.User">
    <!--@mbg.generated-->
    <id column="id" jdbcType="BIGINT" property="id" />
    <result column="name" jdbcType="VARCHAR" property="name" />
    <result column="age" jdbcType="INTEGER" property="age" />
    <result column="email" jdbcType="VARCHAR" property="email" />
  </resultMap>
  <sql id="Base_Column_List">
    <!--@mbg.generated-->
    id, `name`, age, email
  </sql>
  <select id="selectByPrimaryKey" parameterType="java.lang.Long" resultMap="BaseResultMap">
    <!--@mbg.generated-->
    select 
    <include refid="Base_Column_List" />
    from user
    where id = #{id,jdbcType=BIGINT}
  </select>
  <delete id="deleteByPrimaryKey" parameterType="java.lang.Long">
    <!--@mbg.generated-->
    delete from user
    where id = #{id,jdbcType=BIGINT}
  </delete>
  <insert id="insert" keyColumn="id" keyProperty="id" parameterType="com.rigerwu.web.mybatis.User" useGeneratedKeys="true">
    <!--@mbg.generated-->
    insert into user (`name`, age, email
      )
    values (#{name,jdbcType=VARCHAR}, #{age,jdbcType=INTEGER}, #{email,jdbcType=VARCHAR}
      )
  </insert>
  <insert id="insertSelective" keyColumn="id" keyProperty="id" parameterType="com.rigerwu.web.mybatis.User" useGeneratedKeys="true">
    <!--@mbg.generated-->
    insert into user
    <trim prefix="(" suffix=")" suffixOverrides=",">
      <if test="name != null">
        `name`,
      </if>
      <if test="age != null">
        age,
      </if>
      <if test="email != null">
        email,
      </if>
    </trim>
    <trim prefix="values (" suffix=")" suffixOverrides=",">
      <if test="name != null">
        #{name,jdbcType=VARCHAR},
      </if>
      <if test="age != null">
        #{age,jdbcType=INTEGER},
      </if>
      <if test="email != null">
        #{email,jdbcType=VARCHAR},
      </if>
    </trim>
  </insert>
  <update id="updateByPrimaryKeySelective" parameterType="com.rigerwu.web.mybatis.User">
    <!--@mbg.generated-->
    update user
    <set>
      <if test="name != null">
        `name` = #{name,jdbcType=VARCHAR},
      </if>
      <if test="age != null">
        age = #{age,jdbcType=INTEGER},
      </if>
      <if test="email != null">
        email = #{email,jdbcType=VARCHAR},
      </if>
    </set>
    where id = #{id,jdbcType=BIGINT}
  </update>
  <update id="updateByPrimaryKey" parameterType="com.rigerwu.web.mybatis.User">
    <!--@mbg.generated-->
    update user
    set `name` = #{name,jdbcType=VARCHAR},
      age = #{age,jdbcType=INTEGER},
      email = #{email,jdbcType=VARCHAR}
    where id = #{id,jdbcType=BIGINT}
  </update>
</mapper>
```

看，单表的简单增删改查已经帮我们生成好了。

MybatisX 还能让我们快速得在 XML 和 Dao 之间跳转：

![Kapture 2021-01-23 at 21.54.30](https://riger.oss-cn-shanghai.aliyuncs.com/img/Kapture%202021-01-23%20at%2021.54.30.gif)

甚至能根据字段提示，自动帮我们写增删改查的 sql：

![Kapture 2021-01-23 at 22.09.14](https://riger.oss-cn-shanghai.aliyuncs.com/img/Kapture%202021-01-23%20at%2022.09.14.gif)

是不是爽歪歪？

## 5. 测试

我们简单写个测试类，测几个方法：

```java
package com.rigerwu.web.mybatis.mapper;

import com.rigerwu.web.mybatis.entity.User;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.List;

@Slf4j
@SpringBootTest
class UserDaoTest {

    @Autowired
    private UserDao userDao;

    @Test
    void insert() {
        User user = new User();
        user.setName("Tony");
        user.setAge(22);
        user.setEmail("tony@rigerwu.com");
        userDao.insert(user);
        log.info(user.toString());
        Assertions.assertTrue(user.getId() > 0);
    }

    @Test
    void deleteByPrimaryKey() {
        List<User> tony = userDao.queryByNameAndEmail("Tony", "tony@rigerwu.com");
        tony.forEach(user -> {
            int i = userDao.deleteByPrimaryKey(user.getId());
            Assertions.assertTrue(i > 0);
        });
    }

    @Test
    void selectByPrimaryKey() {
        User user = userDao.selectByPrimaryKey(1L);
        log.info(user.toString());
        Assertions.assertEquals((long) user.getId(), 1);
    }

    @Test
    void updateByPrimaryKeySelective() {
        User user = new User();
        user.setId(1L);
        user.setEmail("tom1@rigerwu.com");
        int i = userDao.updateByPrimaryKeySelective(user);
        Assertions.assertTrue( i > 0);
        User updatedUser = userDao.selectByPrimaryKey(1L);
        Assertions.assertTrue(updatedUser.getEmail().startsWith("tom1"));
    }

    @Test
    void queryByNameAndEmail() {
        List<User> jack = userDao.queryByNameAndEmail("Jack", "jack@rigerwu.com");
        Assertions.assertEquals(jack.size(), 1);
    }
}
```

顺利通过测试：

![image-20210124154210845](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210124154210845.png)

## 6. Druid监控

我们使用的是 Druid 数据源，并在配置文件配置了相应监控的配置，接下来测试一下能付正常使用

写一个简单的 Controller：

```java
@RestController
public class UserController {

    // 这里为了演示,直接调的dao,实际开发应当使用service
    @Autowired
    private UserDao userDao;

    @GetMapping("/user/{id}")
    public User getUserById(@PathVariable("id") Long id) {
        return userDao.selectByPrimaryKey(id);
    }
}
```

我们项目启动起来，访问一下我们的 Controller：

![image-20210124160306141](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210124160306141.png)

正常返回了结果，然后登录 druid 监控地址：http://localhost:8090/druid/

![image-20210124155152663](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210124155152663.png)

输入我们设置的账号密码：riger/riger，查看 Sql 监控：

![image-20210124160418924](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210124160418924.png)

可以看到我们的 SQL 执行记录，这在实际生产中可以帮我们分析慢SQL，和查看数据源状态，非常方便。

好了，Mybatis 入门就讲到这里。

源码及脚本都在[Github](https://github.com/RigerWu/web-starter-demos)上

Enjoy it!

















