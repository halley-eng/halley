---
title: "SpringFramework之AutowiredAnnotationBeanPostProcessor"
date: 2020-12-22T21:39:16+08:00
draft: false
tags: ["spring"]
---
## 支持注入的注解类型

从如下构造方法可知其支持的给两种注解类型
1. Autowired
2. Value

```java
	/**
	 * Create a new AutowiredAnnotationBeanPostProcessor
	 * for Spring's standard {@link Autowired} annotation.
	 * <p>Also supports JSR-330's {@link javax.inject.Inject} annotation, if available.
	 */
	@SuppressWarnings("unchecked")
	public AutowiredAnnotationBeanPostProcessor() {
		this.autowiredAnnotationTypes.add(Autowired.class);
		this.autowiredAnnotationTypes.add(Value.class);
		try {
			this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
					ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
			logger.trace("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```


## 注入的时机


Value 注解和 Autowire 都是由 AutowiredAnnotationBeanPostProcessor 负责解析依赖并注入的
但是Value的数据源为需要手动注册 PropertySourcesPlaceholderConfigurer

org.springframework.beans.factory.support.AbstractBeanFactory#resolveEmbeddedValue
