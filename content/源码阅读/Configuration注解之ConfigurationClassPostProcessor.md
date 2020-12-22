---
title: "Configuration注解之ConfigurationClassPostProcessor"
date: 2020-12-22T21:39:16+08:00
draft: false
---




![cbbd87d57e93d3f39aeb42d13b362590.png](evernotecid://0C0C6CA7-E0B1-4D07-A08B-2457E22E1166/appyinxiangcom/2181761/ENResource/p520)


### 运行时机

#### BeanDefinitionRegistryPostProcessor 钩子

通过实现 BeanDefinitionRegistryPostProcessor 接口, 可以在IOC容器刷新阶段的调用, 如下方法

```java
	/**
	 * Derive further bean definitions from the configuration classes in the registry.
	 */
	@Override
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
		// 1. 保证IOC容器只能处理一次;
		int registryId = System.identityHashCode(registry);
		if (this.registriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException(
					"postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
		}
		if (this.factoriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called on this post-processor against " + registry);
		}
		this.registriesPostProcessed.add(registryId);

		// 2. 处理所有的bean定义;
		processConfigBeanDefinitions(registry);
	}

```



#### BeanFactoryPostProcessor 钩子

```java
	/**
	 * Prepare the Configuration classes for servicing bean requests at runtime
	 * by replacing them with CGLIB-enhanced subclasses.
	 */
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		int factoryId = System.identityHashCode(beanFactory);
		if (this.factoriesPostProcessed.contains(factoryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called on this post-processor against " + beanFactory);
		}
		this.factoriesPostProcessed.add(factoryId);
		// 1. 加载并处理IOC容器中Configuration对象: 注册BeanDefinition;
		if (!this.registriesPostProcessed.contains(factoryId)) {
			// BeanDefinitionRegistryPostProcessor hook apparently not supported...
			// Simply call processConfigurationClasses lazily at this point then.
			processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
		}
		// 2. 增强Configuration 对象: 使得任何相关方法的调用都经过代理, 代理会结合注解返回合适的对象;
		enhanceConfigurationClasses(beanFactory);
		// 3. 注册 ImportAwareBeanPostProcessor 使得配置类能够感知到上有配置类;
		beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
	}
```

### 解析并注册BeanDefinition

```java
		// 1. 收集配置类名称
		List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
		String[] candidateNames = registry.getBeanDefinitionNames();

		for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
					ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
			// 收集bean的名称;
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}

		// 如果未收集到则直接短路;
		// Return immediately if no @Configuration classes were found
		if (configCandidates.isEmpty()) {
			return;
		}
		// 2. 将配置类按照优先级 进行排序;
		// Sort by previously determined @Order value, if applicable
		configCandidates.sort((bd1, bd2) -> {
			int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
			int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
			return Integer.compare(i1, i2);
		});

   		// 3. 当前 PostProcessor 引用到 容器中的 BeanNameGenerator
		// Detect any custom bean name generation strategy supplied through the enclosing application context
		SingletonBeanRegistry sbr = null;
		if (registry instanceof SingletonBeanRegistry) {
			sbr = (SingletonBeanRegistry) registry;
			if (!this.localBeanNameGeneratorSet) {
				BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
				if (generator != null) {
					this.componentScanBeanNameGenerator = generator;
					this.importBeanNameGenerator = generator;
				}
			}
		}
		// 4. 初始化环境变量
		if (this.environment == null) {
			this.environment = new StandardEnvironment();
		}

		// 5. 初始化解析器
		// Parse each @Configuration class
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		// 6. 循环解析所有配置类, 这样可以支持配置类中包含配置类这种嵌套场景;
		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		do {
			// 6.1 解析并验证
			parser.parse(candidates);
			parser.validate();

			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);
			// 6.2 初始化BeanDefinitionReader
			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			// 6.3 从配置类解析结果中注册相关 BeanDefinition 到容器中;
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);
			// 6.4 清空候选解析容器, 并从bean容器中筛选可能新的Configuration类, 将其加入candidate中, 等待下一轮迭代;
			candidates.clear();
			if (registry.getBeanDefinitionCount() > candidateNames.length) {
				String[] newCandidateNames = registry.getBeanDefinitionNames();
				Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
				Set<String> alreadyParsedClasses = new HashSet<>();
				for (ConfigurationClass configurationClass : alreadyParsed) {
					alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
				}
				for (String candidateName : newCandidateNames) {
					if (!oldCandidateNames.contains(candidateName)) {
						BeanDefinition bd = registry.getBeanDefinition(candidateName);
						if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
								!alreadyParsedClasses.contains(bd.getBeanClassName())) {
							candidates.add(new BeanDefinitionHolder(bd, candidateName));
						}
					}
				}
				candidateNames = newCandidateNames;
			}
		}
		while (!candidates.isEmpty());

		// 7. 注册当前配置栈
		// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
		if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
			sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
		}

		// 8. 清空metadata缓存;
		if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
			// Clear cache in externally provided MetadataReaderFactory; this is a no-op
			// for a shared cache since it'll be cleared by the ApplicationContext.
			((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
		}
	}
```

主要逻辑为:

1. 收集配置类名称
2. 排序
3. 当前 PostProcessor 引用到 容器中的 BeanNameGenerator
4. 初始化环境变量
5. 初始化解析器
6. 迭代解析所有配置类(支持嵌套)
    1. 解析并验证
    2. 初始化BeanDefinitionReader
    3. 从配置类解析结果中注册相关 BeanDefinition 到容器中
    4. 清空候选解析容器, 并从bean容器中筛选可能新的Configuration类, 将其加入candidate中, 等待下一轮迭代;
7. 注册当前配置栈
8. 清空meta缓存

### 注册相关

AnnotatedBeanDefinitionReader#AnnotatedBeanDefinitionReader(BeanDefinitionRegistry, Environment)
AnnotationConfigUtils#registerAnnotationConfigProcessors(BeanDefinitionRegistry, Object)

在这里会注册

1. ContextAnnotationAutowireCandidateResolver
2. ConfigurationClassPostProcessor
3. AutowiredAnnotationBeanPostProcessor
4. jsr250Present && CommonAnnotationBeanPostProcessor
5. jpaPresent && PersistenceAnnotationBeanPostProcessor
6. EventListenerMethodProcessor
7. DefaultEventListenerFactory


