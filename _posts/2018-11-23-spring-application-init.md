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

### 容器的初始化过程

    我们知道在spring中BeanDefinition很重要。因为Bean的创建是基于它的。容器AbstractApplicationContext中有个方法是refresh()
    这个里面包含了整个Bean的初始化流程实现过程，如果我们需要自己写一个ClassPathXmlApplicationContext类的话我们可以继承这个类，
    下面贴段这个方法的代码：

```
public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            // Prepare this context for refreshing.
            prepareRefresh();

            // Tell the subclass to refresh the internal bean factory.
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // Prepare the bean factory for use in this context.
            prepareBeanFactory(beanFactory);

            try {
                // Allows post-processing of the bean factory in context subclasses.
                postProcessBeanFactory(beanFactory);

                // Invoke factory processors registered as beans in the context.
                invokeBeanFactoryPostProcessors(beanFactory);

                // Register bean processors that intercept bean creation.
                registerBeanPostProcessors(beanFactory);

                // Initialize message source for this context.
                initMessageSource();

                // Initialize event multicaster for this context.
                initApplicationEventMulticaster();

                // Initialize other special beans in specific context subclasses.
                onRefresh();

                // Check for listener beans and register them.
                registerListeners();

                // Instantiate all remaining (non-lazy-init) singletons.
                finishBeanFactoryInitialization(beanFactory);

                // Last step: publish corresponding event.
                finishRefresh();
            }

            catch (BeansException ex) {
                // Destroy already created singletons to avoid dangling resources.
                beanFactory.destroySingletons();

                // Reset 'active' flag.
                cancelRefresh(ex);

                // Propagate exception to caller.
                throw ex;
            }
        }
    }
```

![image](https://dylanblog.github.io/img/in-post/2018-11-23-spring-application-init.jpg)
