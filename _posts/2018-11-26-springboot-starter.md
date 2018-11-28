---
layout: post
title: "Springboot构建自己的Starter"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Spring
  - Springboot
  - Starter
  - spring.factories
  - EnableAutoConfiguration
  - ConditionalOnBean
  - ConditionalOnClass
  - ConditionalOnExpression
  - ConditionalOnJava
  - ConditionalOnJndi
  - ConditionalOnMissingBean
  - ConditionalOnMissingClass
  - ConditionalOnNotWebApplication
  - ConditionalOnProperty
  - ConditionalOnResource
  - ConditionalOnSingleCandidate
  - ConditionalOnWebApplication
---

<a href="https://www.cnblogs.com/leihuazhe/p/7743479.html" target="_blank">参考文章</a>


SpringBoot 自动配置主要通过
> - @EnableAutoConfiguration,
> - @Conditional,
> - @EnableConfigurationProperties
> - 或者 @ConfigurationProperties 等几个注解来进行自动配置完成的。

- @EnableAutoConfiguration 开启自动配置，主要作用就是调用 Spring-Core 包里的 loadFactoryNames()，将 autoconfig 包里的已经写好的自动配置加载进来。
- @Conditional 条件注解，通过判断类路径下有没有相应配置的 jar 包来确定是否加载和自动配置这个类。
- @EnableConfigurationProperties 的作用就是，给自动配置提供具体的配置参数，只需要写在 application.properties 中，就可以通过映射写入配置类的 POJO 属性中。
- @EnableAutoConfiguration
- @Enable*注释并不是SpringBoot新发明的注释，Spring 3框架就引入了这些注释，用这些注释替代XML配置文件。比如：
- @EnableTransactionManagement注释，它能够声明事务管理
- @EnableWebMvc注释，它能启用Spring MVC
- @EnableScheduling注释，它可以初始化一个调度器。

这些注释事实上都是简单的配置，通过@Import注释导入。

从启动类的@SpringBootApplication进入，在里面找到了@EnableAutoConfiguration,

```java
@SpringBootApplication
public class ApplicationMain {

    public static void main(String[] args) {
        SpringApplication.run(ApplicationMain.class, args);
    }
}
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
}
```


@EnableAutoConfiguration里通过@Import导入了EnableAutoConfigurationImportSelector,

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	String[] excludeName() default {};

}
```

进入他的父类AutoConfigurationImportSelector

```java
public class EnableAutoConfigurationImportSelector
		extends AutoConfigurationImportSelector {

	@Override
	protected boolean isEnabled(AnnotationMetadata metadata) {
		if (getClass().equals(EnableAutoConfigurationImportSelector.class)) {
			return getEnvironment().getProperty(
					EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY, Boolean.class,
					true);
		}
		return true;
	}
}
```

找到selectImports()方法，他调用了getCandidateConfigurations()方法，在这里，这个方法又调用了Spring Core包中的loadFactoryNames()方法。这个方法的作用是，会查询META-INF/spring.factories文件中包含的JAR文件。

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return NO_IMPORTS;
	}
	try {
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
		List<String> configurations = getCandidateConfigurations(annotationMetadata,
				attributes);
		configurations = removeDuplicates(configurations);
		configurations = sort(configurations, autoConfigurationMetadata);
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
		configurations = filter(configurations, autoConfigurationMetadata);
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return configurations.toArray(new String[configurations.size()]);
	}
	catch (IOException ex) {
		throw new IllegalStateException(ex);
	}
}
```
当找到spring.factories文件后，SpringFactoriesLoader将查询配置文件命名的属性。

```java
/**
 * Return the auto-configuration class names that should be considered. By default
 * this method will load candidates using {@link SpringFactoriesLoader} with
 * {@link #getSpringFactoriesLoaderFactoryClass()}.
 * @param metadata the source metadata
 * @param attributes the {@link #getAttributes(AnnotationMetadata) annotation
 * attributes}
 * @return a list of candidate configurations
 */
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
		AnnotationAttributes attributes) {
	List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
			getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
	Assert.notEmpty(configurations,
			"No auto configuration classes found in META-INF/spring.factories. If you "
					+ "are using a custom packaging, make sure that file is correct.");
	return configurations;
}
```

```java
/**
 * Load the fully qualified class names of factory implementations of the
 * given type from {@value #FACTORIES_RESOURCE_LOCATION}, using the given
 * class loader.
 * @param factoryClass the interface or abstract class representing the factory
 * @param classLoader the ClassLoader to use for loading resources; can be
 * {@code null} to use the default
 * @see #loadFactories
 * @throws IllegalArgumentException if an error occurs while loading factory names
 */
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
	String factoryClassName = factoryClass.getName();
	try {
		Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
				ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
		List<String> result = new ArrayList<String>();
		while (urls.hasMoreElements()) {
			URL url = urls.nextElement();
			Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
			String factoryClassNames = properties.getProperty(factoryClassName);
			result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
		}
		return result;
	}
	catch (IOException ex) {
		throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
				"] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
	}
}

/**
 * The location to look for factories.
 * <p>Can be present in multiple JAR files.
 */
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
```

Jar文件在org.springframework.boot.autoconfigure的spring.factories

spring.factories内容如下(截取部分),在这个文件中，可以看到一系列Spring Boot自动配置的列表

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.ldap.LdapDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.ldap.LdapRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.solr.SolrRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.rest.RepositoryRestMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration,\
```

下面我们来看自动配置redis的细节，RedisAutoConfiguration：
RedisAutoConfiguration
redis.png

这个类进行了简单的Spring配置，声明了Redis所需典型Bean，和其它很多类一样，重度依赖于Spring Boot注释：
- @ConditionOnClass 激活一个配置，当类路径中存在这个类时才会配置该类
- @EnableConfigurationProperties 自动映射一个POJO到Spring Boot配置文件（默认是application.properties文件）的属性集。
- @ConditionalOnMissingBean 启用一个Bean定义，但必须是这个Bean之前未定义过才有效。

还可以使用@ AutoConfigureBefore注释、@AutoConfigureAfter注释来定义这些配置类的载入顺序。

着重了解@Conditional注释，Spring 4框架的新特性
此注释使得只有在特定条件满足时才启用一些配置。SrpingBoot的AutoConfig大量使用了@Conditional，它会根据运行环境来动态注入Bean。
这里介绍一些@Conditional的使用和原理，并自定义@Conditional来自定义功能。

@Conditional 是SpringFramework的功能，SpringBoot在它的基础上定义了
@ConditionalOnClass，@ConditionalOnProperty等一系列的注解来实现更丰富的内容。

具体几个@Conditon*注解的含义
> - @ConditionalOnBean 仅仅在当前上下文中存在某个对象时，才会实例化一个Bean
> - @ConditionalOnClass 某个class位于类路径上，才会实例化一个Bean)，该注解的参数对应的类必须存在，否则不解析该注解修饰的配置类
> - @ConditionalOnExpression 当表达式为true的时候，才会实例化一个Bean
> - @ConditionalOnMissingBean 仅仅在当前上下文中不存在某个对象时，才会实例化一个Bean，该注解表示，如果存在它修饰的类的bean，则不需要再创建这个bean，可以给该注解传入参数例如@ConditionOnMissingBean(name = "example")，这个表示如果name为“example”的bean存在，这该注解修饰的代码块不执行
> - @ConditionalOnMissingClass 某个class类路径上不存在的时候，才会实例化一个Bean
> - @ConditionalOnNotWebApplication 不是web应用时，才会执行

Properties系列注释

- @EnableConfigurationProperties
- @ConfigurationProperties(prefix = "may")

在需要注入配置的类上加上这个注解，prefix的意思是，以该前缀打头的配置，以下是例子
```
    @ConfigurationProperties(prefix = "may")
    public class User {
        private String name;
        private String gender;

       //省略setter,getter方法
    }
 ```
application.yml中的配置

```
   may
      name: youjie
      gender: man
```

如果不用系统初始的application.yml配置类，而是使用自己的如youjie.yml，可以如下配置

```
    @ConfigurationProperties(prefix = "may",locations = "classpath:youjie.yml")
    public class User2 {
        private String name;
        private String gender;

       //省略setter,getter方法

    }
```

过时：由于Spring-boot 1.5.2版本移除了，locations这个属性,因此上述这种方式在最新的版本中过时。

```
@PropertySource
```

Spring-boot 1.5.2版本之后，采用下面这种方式

```
@Component
//@PropertySource只能加载.properties文件，需要将上面的yml文件，改为.properties文件
@PropertySource("classpath:may.properties")
@ConfigurationProperties(prefix="may")
public class User2 {
    private String name;
    private String gender;
   //省略setter,getter方法
}
```

@EnableConfigurationProperties

最后注意在spring Boot入口类加上@EnableConfigurationProperties

```
@SpringBootApplication
@EnableConfigurationProperties({User.class,User2.class})
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```
其实这里@EnableConfigurationProperties({User.class,User2.class}) 可以省略

config.png

总结

SpringBoot 的 自动配置得益于 SpringFramework 强大的支撑，框架早已有很多工具和注解可以自动装配 Bean 。SpringBoot 通过 一个封装，
将市面上通用的组件直接写好了配置类。当我们程序去依赖了这些组件的 jar 包后，启动 SpringBoot应用，于是自动加载开始了。

我们也可以定义自己的自动装配组件，依赖之后，Spring直接可以加载我们定义的 starter :

#### 1. 创建自动配置类

```java
package com.chhliu.springboot.starter.helloworld;

import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * 描述：人员信息自动配置属性类
 * @author chhliu
 * 创建时间：2017年2月13日 下午9:05:34
 * @version 1.2.0
 */
@ConfigurationProperties(prefix="person.proterties.set")// 定义application.properties配置文件中的配置前缀
public class PersonServiceProperties {

	// 姓名
	private String name;
	// 年龄
	private int age;
	// 性别，不配置的时候默认为person.proterties.set=man
	private String sex = "man";
	// 身高
	private String height;
	// 体重
	private String weight;

	……省略getter，setter方法……
}
```

#### 2.编写服务类

```java
package com.chhliu.springboot.starter.helloworld.service;

import com.chhliu.springboot.starter.helloworld.PersonServiceProperties;

public class PersonService {

	private PersonServiceProperties properties;

	public PersonService(PersonServiceProperties properties){
		this.properties = properties;
	}

	public PersonService(){

	}

	public String getPersonName(){
		return properties.getName();
	}

	public int getPersonAge(){
		return properties.getAge();
	}

	public String getPersonSex(){
		return properties.getSex();
	}
```

#### 3.自动配置类

```java
package com.chhliu.springboot.starter.helloworld.configuration;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.chhliu.springboot.starter.helloworld.PersonServiceProperties;
import com.chhliu.springboot.starter.helloworld.service.PersonService;

@Configuration // 配置注解
@EnableConfigurationProperties(PersonServiceProperties.class) // 开启指定类的配置
@ConditionalOnClass(PersonService.class)// 当PersonService这个类在类路径中时，且当前容器中没有这个Bean的情况下，开始自动配置
@ConditionalOnProperty(prefix="person.proterties.set", value="enabled", matchIfMissing=true)// 指定的属性是否有指定的值
public class PersonServiceAutoConfiguration {
	@Autowired
	private PersonServiceProperties properties;

	@Bean
  	@ConditionalOnMissingBean(PersonService.class)// 当容器中没有指定Bean的情况下，自动配置PersonService类
	public PersonService personService(){
		PersonService personService = new PersonService(properties);
		return personService;
	}
}
```
#### 4.注册配置

1. 在src/main/resources新建META-INF文件夹
2. 在META-INF文件夹下新建spring.factories文件

**注意：spring.factories并非是必须的，可以在启动类上添加如下注解进行自动配置**
```java
@ImportAutoConfiguration({PersonServiceAutoConfiguration.class})

```
3. 在spring.factories文件注册配置自动配置类

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.chhliu.springboot.starter.helloworld.configuration.PersonServiceAutoConfiguration
```

#### 5.延伸知识

```java
@ConditionalOnBean: 当容器中有指定的Bean的条件下
@ConditionalOnClass: 当类路径下有指定的类的条件下
@ConditionalOnExpression: 基于SpEL表达式作为判断条件
@ConditionalOnJava: 基于JVM版本作为判断条件
@ConditionalOnJndi: 在JNDI存在的条件下查找指定的位置
@ConditionalOnMissingBean: 当容器中没有指定Bean的情况下
@ConditionalOnMissingClass: 当类路径下没有指定的类的条件下
@ConditionalOnNotWebApplication: 当前项目不是Web项目的条件下
@ConditionalOnProperty: 指定的属性是否有指定的值
@ConditionalOnResource: 类路径下是否有指定的资源
@ConditionalOnSingleCandidate: 当指定的Bean在容器中只有一个，或者在有多个Bean的情况下，用来指定首选的Bean
@ConditionalOnWebApplication: 当前项目是Web项目的条件下
```