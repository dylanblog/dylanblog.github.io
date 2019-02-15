---
layout: post
title: "Spring Boot日志框架实践"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Springboot
  - 日志
---

<a href="http://www.open-open.com/lib/view/open1522307963845.html" target="_blank">原文地址</a>

#### 概述

Java应用中，日志一般分为以下5个级别：

- ERROR 错误信息
- WARN 警告信息
- INFO 一般信息
- DEBUG 调试信息
- TRACE 跟踪信息

Spring Boot使用Apache的Commons Logging作为内部的日志框架，其仅仅是一个日志接口，在实际应用中需要为该接口来指定相应的日志实现。

SpringBt默认的日志实现是Java Util Logging，是JDK自带的日志包，此外SpringBt当然也支持Log4J、Logback这类很流行的日志实现。

统一将上面这些 日志实现 统称为 日志框架

下面我们来实践一下！

使用Spring Boot Logging插件
首先application.properties文件中加配置：

```
logging.level.root=INFO
```

控制器部分代码如下：

```
import com.hansonwang99.K8sresctrlApplication;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/testlogging")
public class LoggingTestController {
    private static Logger logger = LoggerFactory.getLogger(K8sresctrlApplication.class);
    @GetMapping("/hello")
    public String hello() {
        logger.info("test logging...");
        return "hello";
    }
}
```

运行结果


由于将日志等级设置为INFO，因此包含INFO及以上级别的日志信息都会打印出来

这里可以看出，很多大部分的INFO日志均来自于SpringBt框架本身，如果我们想屏蔽它们，可以将日志级别统一先全部设置为ERROR，这样框架自身的INFO信息不会被打印。然后再将应用中特定的包设置为DEBUG级别的日志，这样就可以只看到所关心的包中的DEBUG及以上级别的日志了。

控制特定包的日志级别
application.yml中改配置

```
logging:
  level:
    root: error
    com.hansonwang99.controller: debug
```

很明显，将root日志级别设置为ERROR，然后再将 com.hansonwang99.controller 包的日志级别设为DEBUG，此即：即先禁止所有再允许个别的 设置方法

控制器代码

```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/testlogging")
public class LoggingTestController {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    @GetMapping("/hello")
    public String hello() {
        logger.info("test logging...");
        return "hello";
    }
}
```

运行结果

可见框架自身的INFO级别日志全部藏匿，而指定包中的日志按级别顺利地打印出来

将日志输出到某个文件中

```
logging:
  level:
    root: error
    com.hansonwang99.controller: debug
  file: ${user.home}/logs/hello.log
```
  
运行结果


使用Spring Boot Logging，我们发现虽然日志已输出到文件中，但控制台中依然会打印一份，发现用 org.slf4j.Logger 是无法解决这个问题的

集成Log4J日志框架
pom.xml中添加依赖

```
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-log4j2</artifactId>
        </dependency>
```

在resources目录下添加 log4j2.xml 文件，内容如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <appenders>
        <File name="file" fileName="${sys:user.home}/logs/hello2.log">
            <PatternLayout pattern="%d{HH:mm:ss,SSS} %p %c (%L) - %m%n"/>
        </File>
    </appenders>

    <loggers>

        <root level="ERROR">
            <appender-ref ref="file"/>
        </root>
        <logger name="com.hansonwang99.controller" level="DEBUG" />
    </loggers>

</configuration>
```

其他代码都保持不变
运行程序发现控制台没有日志输出，而hello2.log文件中有内容，这符合我们的预期：


而且日志格式和 pattern="%d{HH:mm:ss,SSS} %p %c (%L) - %m%n" 格式中定义的相匹配

Log4J更进一步实践
pom.xml配置：

```
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-log4j2</artifactId>
        </dependency>
 ```
 
log4j2.xml配置

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration status="warn">
    <properties>

        <Property name="app_name">springboot-web</Property>
        <Property name="log_path">logs/${app_name}</Property>

    </properties>
    <appenders>
        <console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="[%d][%t][%p][%l] %m%n" />
        </console>

        <RollingFile name="RollingFileInfo" fileName="${log_path}/info.log"
                     filePattern="${log_path}/$${date:yyyy-MM}/info-%d{yyyy-MM-dd}-%i.log.gz">
            <Filters>
                <ThresholdFilter level="INFO" />
                <ThresholdFilter level="WARN" onMatch="DENY"
                                 onMismatch="NEUTRAL" />
            </Filters>
            <PatternLayout pattern="[%d][%t][%p][%c:%L] %m%n" />
            <Policies>
                <!-- 归档每天的文件 -->
                <TimeBasedTriggeringPolicy interval="1" modulate="true" />
                <!-- 限制单个文件大小 -->
                <SizeBasedTriggeringPolicy size="2 MB" />
            </Policies>
            <!-- 限制每天文件个数 -->
            <DefaultRolloverStrategy compressionLevel="0" max="10"/>
        </RollingFile>

        <RollingFile name="RollingFileWarn" fileName="${log_path}/warn.log"
                     filePattern="${log_path}/$${date:yyyy-MM}/warn-%d{yyyy-MM-dd}-%i.log.gz">
            <Filters>
                <ThresholdFilter level="WARN" />
                <ThresholdFilter level="ERROR" onMatch="DENY"
                                 onMismatch="NEUTRAL" />
            </Filters>
            <PatternLayout pattern="[%d][%t][%p][%c:%L] %m%n" />
            <Policies>
                <!-- 归档每天的文件 -->
                <TimeBasedTriggeringPolicy interval="1" modulate="true" />
                <!-- 限制单个文件大小 -->
                <SizeBasedTriggeringPolicy size="2 MB" />
            </Policies>
            <!-- 限制每天文件个数 -->
            <DefaultRolloverStrategy compressionLevel="0" max="10"/>
        </RollingFile>

        <RollingFile name="RollingFileError" fileName="${log_path}/error.log"
                     filePattern="${log_path}/$${date:yyyy-MM}/error-%d{yyyy-MM-dd}-%i.log.gz">
            <ThresholdFilter level="ERROR" />
            <PatternLayout pattern="[%d][%t][%p][%c:%L] %m%n" />
            <Policies>
                <!-- 归档每天的文件 -->
                <TimeBasedTriggeringPolicy interval="1" modulate="true" />
                <!-- 限制单个文件大小 -->
                <SizeBasedTriggeringPolicy size="2 MB" />
            </Policies>
            <!-- 限制每天文件个数 -->
            <DefaultRolloverStrategy compressionLevel="0" max="10"/>
        </RollingFile>

    </appenders>

    <loggers>


        <root level="info">
            <appender-ref ref="Console" />
            <appender-ref ref="RollingFileInfo" />
            <appender-ref ref="RollingFileWarn" />
            <appender-ref ref="RollingFileError" />
        </root>

    </loggers>

</configuration>
```

控制器代码：

```
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/testlogging")
public class LoggingTestController {
    private final Logger logger = LogManager.getLogger(this.getClass());
    @GetMapping("/hello")
    public String hello() {
        for(int i=0;i<10_0000;i++){
            logger.info("info execute index method");
            logger.warn("warn execute index method");
            logger.error("error execute index method");
        }
        return "My First SpringBoot Application";
    }
}
```

运行结果

日志会根据不同的级别存储在不同的文件，当日志文件大小超过2M以后会分多个文件压缩存储，生产环境的日志文件大小建议调整为20-50MB。