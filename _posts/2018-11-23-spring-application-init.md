---
layout: post
title: "Spring Bean初始化过程"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Spring
  - ApplicationContext
  -
---


<a href="https://blog.csdn.net/he90227/article/details/51481364" target="_blank">参考文章</a>

### 容器的初始化过程

    我们知道在spring中BeanDefinition很重要。因为Bean的创建是基于它的。容器AbstractApplicationContext中有个方法是refresh()
    这个里面包含了整个Bean的初始化流程实现过程，如果我们需要自己写一个ClassPathXmlApplicationContext类的话我们可以继承这个类，
    下面贴段这个方法的代码：

```
//加载或刷新持久的配置，可能是xml文件，properties文件，或者关系型数据库的概要。  
//做为一个启动方法，如果初始化失败将会销毁已经创建好的单例，避免重复加载配置文件。  
//换句话说，在执行这个方法之后，要不全部加载单例，要不都不加载  
public void refresh() throws BeansException, IllegalStateException   
{  
    synchronized (this.startupShutdownMonitor)   
    {  
        // 初始化配置准备刷新,验证环境变量中的一些必选参数  
        prepareRefresh();  
  
        // 告诉继承类销毁内部的factory创建新的factory的实例  
        // 初始化Bean实例  
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
  
        // 初始化beanFactroy的基本信息，包括classloader，environment，忽略的注解等  
        prepareBeanFactory(beanFactory);  
  
        try {  
            // beanfactory内部的postProcess，可以理解为context中PostProcess的补充  
            beanFactory.postProcessBeanFactory(beanFactory);  
  
            // 执行BeanFactoryPostProcessor（在beanFactory初始化过程中，bean初始化之前，修改beanfactory参数）、  
            // BeanDefinitionRegistryPostProcessor 其实也是继承自BeanFactoryPostProcessor，  
            // 多了对BeanDefinitionRegistry的支持invokeBeanFactoryPostProcessors(beanFactory);  
            // 执行postProcess，那BeanPostProcessor是什么呢，是为了在bean加载过程中修改bean的内容，  
            // 使用分的有两个而方法Before、After分别对应初始化前和初始化后  
            registerBeanPostProcessors(beanFactory);  
  
            // 初始化MessageSource,主要用作I18N本地化的内容  
            initMessageSource();  
  
            // 初始化事件广播ApplicationEventMulticaster，使用观察者模式，对注册的ApplicationEvent时间进行捕捉  
            initApplicationEventMulticaster();  
  
            // 初始化特殊bean的方法  
            onRefresh();  
  
            // 将所有ApplicationEventListener注册到ApplicationEventMulticaster中  
            registerListeners();  
  
            // 初始化所有不为lazy-init的bean，singleton实例  
            finishBeanFactoryInitialization(beanFactory);  
  
            // 初始化lifecycle的bean并启动（例如quartz的定时器等），如果开启JMX则将ApplicationContext注册到上面  
            finishRefresh();  
        }  
        catch (BeansException ex)   
        {  
            //销毁已经创建单例  
            resources.destroyBeans();  
  
            // 将context的状态转换为无效，标示初始化失败  
            flag.cancelRefresh(ex);  
  
            // 将异常传播到调用者  
            throw ex;  
        }  
    }  
}  
```

![image](https://dylanblog.github.io/img/in-post/2018-11-23-spring-application-init.jpg)
