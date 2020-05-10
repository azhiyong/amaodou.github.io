---
title: Spring Boot启动过程分析
date: 2020-05-09 11:04:39
tags: "Spring Boot"
---

## Spring Boot 应用启动入口

```java
@SpringBootApplication
public class WebApplication {
    public static void main(String[] args) {
        SpringApplication.run(WebApplication.class, args);
    }
}
```

<!--more-->

启动入口类有两个关键元素：`@SpringBootApplication`和`SpringApplication::run`

- `@SpringBootApplication`

  这是由多个注解组合成的一个新的注解，作用等效于使用 `@EnableAutoConfiguration` + `@SpringBootConfiguration` + `@ComponentScan`

- `SpringApplication::run`

  调用`SpringApplication`类的静态`run()`方法，实际上是先实例化了`SpringApplication`对象，然后再调用该对象的`run()`方法。

### 实例化 SpringApplication

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    setInitializers((Collection) getSpringFactoriesInstances(
            ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

实例化`SpringApplication`过程做了 4 件事：推断应用类型、设置 Initializers、设置 Listeners、推断`main`方法定义类

1. 推断应用类型，根据默认类加载器（即 sun.misc.Launcher\$AppClassLoader）是否可以加载指定的类判断应用是 REACTIVE 应用、SERVLET 应用、NONE；

   ```java
   static WebApplicationType deduceFromClasspath() {
       if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
               && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
               && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
           return WebApplicationType.REACTIVE;
       }
       for (String className : SERVLET_INDICATOR_CLASSES) {
           if (!ClassUtils.isPresent(className, null)) {
               return WebApplicationType.NONE;
           }
       }
       return WebApplicationType.SERVLET;
   }
   ```

2. 设置 Initializers，使用默认的类加载器加载 classpath 下的`META-INF/spring.factories`资源文件，解析获得文件中配置的`ApplicationContextInitializer`的实现类并实例化这些类；

   ```java
   private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
       return getSpringFactoriesInstances(type, new Class<?>[] {});
   }

   private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
           Class<?>[] parameterTypes, Object... args) {
       ClassLoader classLoader = getClassLoader();

       // Use names and ensure unique to protect against duplicates
       // 加载spring.factories资源文件，解析获得其中配置的ApplicationContextInitializer类
       Set<String> names = new LinkedHashSet<>(
               SpringFactoriesLoader.loadFactoryNames(type, classLoader));

       // 初始化所有ApplicationContextInitializer类
       List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
               classLoader, args, names);
       AnnotationAwareOrderComparator.sort(instances);
       return instances;
   }

   private <T> List<T> createSpringFactoriesInstances(Class<T> type,
           Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
           Set<String> names) {
       List<T> instances = new ArrayList<>(names.size());
       for (String name : names) {
           try {
               Class<?> instanceClass = ClassUtils.forName(name, classLoader);
               Assert.isAssignable(type, instanceClass);
               Constructor<?> constructor = instanceClass
                       .getDeclaredConstructor(parameterTypes);

               // 通过反射机制类的构造函数创建对象
               T instance = (T) BeanUtils.instantiateClass(constructor, args);
               instances.add(instance);
           }
           catch (Throwable ex) {
               throw new IllegalArgumentException(
                       "Cannot instantiate " + type + " : " + name, ex);
           }
       }
       return instances;
   }
   ```

   `META-INF/spring.factories`配置的`ApplicationContextInitializer`如下：

   ```java
   org.springframework.context.ApplicationContextInitializer=\
   org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
   org.springframework.boot.context.ContextIdApplicationContextInitializer,\
   org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
   org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
   ```

3. 设置 Listeners，实例化`META-INF/spring.factories`中的`ApplicationListener`类；

   `META-INF/spring.factories`配置的`ApplicationListener`如下：

   ```java
   org.springframework.context.ApplicationListener=\
   org.springframework.boot.ClearCachesApplicationListener,\
   org.springframework.boot.builder.ParentContextCloserApplicationListener,\
   org.springframework.boot.context.FileEncodingApplicationListener,\
   org.springframework.boot.context.config.AnsiOutputApplicationListener,\
   org.springframework.boot.context.config.ConfigFileApplicationListener,\
   org.springframework.boot.context.config.DelegatingApplicationListener,\
   org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
   org.springframework.boot.context.logging.LoggingApplicationListener,\
   org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
   ```

4. 推断`main`方法定义类

   ```java
   private Class<?> deduceMainApplicationClass() {
       try {
           StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
           for (StackTraceElement stackTraceElement : stackTrace) {
               if ("main".equals(stackTraceElement.getMethodName())) {
                   return Class.forName(stackTraceElement.getClassName());
               }
           }
       }
       catch (ClassNotFoundException ex) {
           // Swallow and continue
       }
       return null;
   }
   ```

### SpringApplication 的 run()方法

```java
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();

    // 设置java.awt.headless模式
    configureHeadlessProperty();

    // 实例化spring.factories中配置的SpringApplicationRunListener类，即EventPublishingRunListener
    SpringApplicationRunListeners listeners = getRunListeners(args);

    // 发布Starting事件，listeners接收事件并执行相应操作
    listeners.starting();
    try {
        // 封装参数准备环境Environment
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(
                args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners,
                applicationArguments);

        configureIgnoreBeanInfo(environment);
        Banner printedBanner = printBanner(environment);

        // 根据不同的应用类型创建ApplicationContext
        context = createApplicationContext();

        // 实例化spring.factories中配置的SpringBootExceptionReporter类，即FailureAnalyzers
        // FailureAnalyzers用于故障分析并将诊断信息展示给用户
        exceptionReporters = getSpringFactoriesInstances(
                SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);

        // ApplicationContext设置环境，注册类型转换器
        // 使用Initializer初始化ApplicationContext，发布Initialized和Prepared事件
        prepareContext(context, environment, listeners, applicationArguments,
                printedBanner);

        // 调用AbstractApplicationContext的refresh方法，注册Shutdown钩子方法
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass)
                    .logStarted(getApplicationLog(), stopWatch);
        }

        // 发布Started事件
        listeners.started(context);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // 发布Ready事件
        listeners.running(context);
    }
    catch (Throwable ex) {
        // ApplicationContext启动失败发布Failed事件，报告启动失败原因
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

1. 准备 ApplicationContext 环境 Environment

   ```java
   private ConfigurableEnvironment prepareEnvironment(
           SpringApplicationRunListeners listeners,
           ApplicationArguments applicationArguments) {
       // Create and configure the environment
       // 根据应用类型创建Environment
       // SERVLET: StandardServletEnvironment
       // REACTIVE: StandardReactiveWebEnvironment
       // NONE: StandardEnvironment
       ConfigurableEnvironment environment = getOrCreateEnvironment();

       // 初始化ApplicationConversionService并配置类型转换器 (如StringToBooleanConverter, StringToArrayConverter);
       // 配置Environment的PropertySource
       // 配置Environment的Profile, 即加载spring.profiles.active文件中的配置
       configureEnvironment(environment, applicationArguments.getSourceArgs());

       // Environment准备就绪创建ApplicationContext之前，发布ApplicationEnvironmentPreparedEvent事件
       listeners.environmentPrepared(environment);

       // 把Environment绑定到SpringApplication
       bindToSpringApplication(environment);

       if (!this.isCustomEnvironment) {
           environment = new EnvironmentConverter(getClassLoader())
                   .convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
       }
       ConfigurationPropertySources.attach(environment);
       return environment;
   }
   ```

2. 准备 ApplicationContext

   ```java
   private void prepareContext(ConfigurableApplicationContext context,
           ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
           ApplicationArguments applicationArguments, Banner printedBanner) {
       context.setEnvironment(environment);

       // 注册类型转换器
       postProcessApplicationContext(context);
       applyInitializers(context);

       // ApplicationContext创建sources加载前，发布ApplicationContextInitializedEvent事件
       listeners.contextPrepared(context);
       if (this.logStartupInfo) {
           logStartupInfo(context.getParent() == null);
           logStartupProfileInfo(context);
       }
       // Add boot specific singleton beans
       ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
       beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
       if (printedBanner != null) {
           beanFactory.registerSingleton("springBootBanner", printedBanner);
       }
       if (beanFactory instanceof DefaultListableBeanFactory) {
           ((DefaultListableBeanFactory) beanFactory)
                   .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
       }
       // Load the sources
       Set<Object> sources = getAllSources();
       Assert.notEmpty(sources, "Sources must not be empty");
       load(context, sources.toArray(new Object[0]));

       // ApplicationContext加载完刷新前，发布ApplicationPreparedEvent事件
       listeners.contextLoaded(context);
   }
   ```

3. `AbstractApplicationContext`的`refresh()`方法

   ```java
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
               if (logger.isWarnEnabled()) {
                   logger.warn("Exception encountered during context initialization - " +
                           "cancelling refresh attempt: " + ex);
               }

               // Destroy already created singletons to avoid dangling resources.
               destroyBeans();

               // Reset 'active' flag.
               cancelRefresh(ex);

               // Propagate exception to caller.
               throw ex;
           }

           finally {
               // Reset common introspection caches in Spring's core, since we
               // might not ever need metadata for singleton beans anymore...
               resetCommonCaches();
           }
       }
   }
   ```
