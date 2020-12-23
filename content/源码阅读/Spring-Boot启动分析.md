---
title: "Spring-Boot启动分析"
date: 2020-12-22T21:39:16+08:00
draft: false
tags: ["spring-boot"]

---

以常见的SpringBoot 工程启动类举例

```java
@SpringBootApplication(scanBasePackages = {"com.xxx.xxx.xxx"})
@EnableAspectJAutoProxy
@EnableConfigCenter
@EnableMbean
@EnableScheduling
public class FcXxxApplication {

  public static void main(String[] args) throws Exception {
    SpringApplication.run(FcXxxApplication.class, args);
    Runtime.getRuntime().addShutdownHook(new AppShutdownHook());
  }

}
```

这里简单的通过 SpringApplication#run 就可以启动Spring 容器, 并让其运行在 Web 容器中;

下面是其核心源码

```java
public ConfigurableApplicationContext run(String... args) {
		// 1. 计数器初始化并启动
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		// 2. 加载并启动监听器
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			// 3. 初始化环境变量
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			// 4. 创建上下文
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			// 5. 准备上下文 	  1. 初始化容器  		2. 加载容器资源
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
			// 6. 刷新上下文
			refreshContext(context);
			// 7. 后置刷新上下文
			afterRefresh(context, applicationArguments);
			// 8. 停止计数器
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```

其中如下图所示正常流程分成七个不同的阶段:

1. starting : 监听器加载后, 通知启动中;
2. environmentPrepared: 加载完环境变量
3. contextPrepared： 应用完上下文初始化器
4. contextLoaded ： 加载完成spring的bean
5. started: 刷新完容器，已经初始化完所有的容器
6. runding: 前五步没有任何异常
7. failed: 前五步有异常

![697e5af79a53492b26a1eb172a284ae6.png](evernotecid://0C0C6CA7-E0B1-4D07-A08B-2457E22E1166/appyinxiangcom/2181761/ENResource/p519)


```
![20201223182413](https://halley-image.oss-cn-beijing.aliyuncs.com/picgo20201223182413.png)

```

其中比较复杂的是启动上下文的流程:

```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
		context.setEnvironment(environment);
		// 1. 配置 beanNameGenerator、 resourceLoader、 addConversionService
		postProcessApplicationContext(context);
		// 2. 初始化上下文, 并通知上下文准备完毕;
		applyInitializers(context);
		listeners.contextPrepared(context);

		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}
		// 3. 注册启动相关的bean: springApplicationArguments、springBootBanner、LazyInitializationBeanFactoryPostProcessor
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
		//  根据懒加载过滤器决定, 过滤掉某些单例bean, 使得其在启动后在加载;
		if (this.lazyInitialization) {
			context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
		}
		// 4. 加载资源(Bean元数据)并会输出到容器中, 并通知容器加载完毕
		// Load the sources
		Set<Object> sources = getAllSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		load(context, sources.toArray(new Object[0]));

		listeners.contextLoaded(context);
	}
```

### 总结

SpringApplication#run 方法主要负责容器环境准备过程中的7个不同阶段;
