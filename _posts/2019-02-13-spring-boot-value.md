---
layout: post
title: "Value 表达式"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Spring
  - Springboot
  - Value
---


```java
package config;

import org.apache.commons.io.IOUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.*;
import org.springframework.core.env.Environment;
import org.springframework.core.io.Resource;
import repository.dao.UserDAO;
import service.DemoService;
import service.FunctionService;
import service.UseFunctionService;

import java.io.IOException;


@Configuration
@ComponentScan(basePackages = "main.java.com.example.demo")
@PropertySource(value={"test.properties"})
public class SpringConfig {

    @Bean // 通过该注解来表明是一个Bean对象，相当于xml中的<bean>
    public DemoService demoService(){
        return new DemoService(); // 直接new对象做演示
    }

    @Value("注入普通字符串")// 注入普通字符串
    private String normal;

    @Value("#{systemProperties['os.name']}")// 注入操作系统属性
    private String osName;

    @Value("#{T(java.lang.Math).random() * 100.0 }")// 注入表达式结果
    private double randomNumber;

    @Value("#{demoService.another}")// 注入其他Bean属性
    private String fromAnother;

    @Value("classpath:test.txt")// 注入文件资源
    private Resource testFile;

    @Value("https://www.baidu.com")// 注入网址资源
    private Resource testUrl;

    @Value("${book.name}")// 注入配置文件【注意是$符号】
    private String bookName;

    @Autowired// Properties可以从Environment获得
    private Environment environment;//属性文件

    @Override
    public String toString() {
        try {
            return "ELConfig [normal=" + normal
                    + ", osName=" + osName
                    + ", randomNumber=" + randomNumber
                    + ", fromAnother=" + fromAnother
                    + ", testFile=" + IOUtils.toString(testFile.getInputStream())
                    + ", testUrl=" + IOUtils.toString(testUrl.getInputStream())
                    + ", bookName=" + bookName
                    + ", environment=" + environment.getProperty("book.name") + "]";
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
    }
}
```