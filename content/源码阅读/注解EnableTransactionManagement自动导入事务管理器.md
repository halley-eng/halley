---
title: "注解EnableTransactionManagement自动导入事务管理器"
date: 2020-12-22T21:39:16+08:00
draft: false
tags: ["spring"]
---

事务注解的解析流程

1. ConfigurationClassPostProcessor 解析配置类
2. 发现注解类型 EnableTransactionManagement
3. 根据注解找到其对应的 Selector 实现 TransactionManagementConfigurationSelector
4. 解析器调用 selector 实现父类 AdviceModeImportSelector 的 
    selectImports(AnnotationMetadata importingClassMetadata)方法
5. AdviceModeImportSelector 解析出通知模式后 调用selector实现类 的 
    selectImports(AdviceMode adviceMode)

第四部源码如下: 

根据通知模式模式(AspectJ/Proxy)路由不同的配置类;
AdviceModeImportSelector#selectImports(AnnotationMetadata)

```java

	@Override
	public final String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // 1. 查询当前类作为别人的父类是, 子类给的泛型参数是什么, 它是当前类要处理的注解类型;
		Class<?> annoType = GenericTypeResolver.resolveTypeArgument(getClass(), AdviceModeImportSelector.class);
        // 2. 从源配置类中查询出来, 注解参数实际对象;
		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(importingClassMetadata, annoType);
		if (attributes == null) {
			throw new IllegalArgumentException(String.format(
				"@%s is not present on importing class '%s' as expected",
				annoType.getSimpleName(), importingClassMetadata.getClassName()));
		}
        // 3. 获取通知类型  代理/Aspect
		AdviceMode adviceMode = attributes.getEnum(this.getAdviceModeAttributeName());
        // 4. 从实现类中查询出不同通知类型下 应该导入的配置类
		String[] imports = selectImports(adviceMode);
		if (imports == null) {
			throw new IllegalArgumentException(String.format("Unknown AdviceMode: '%s'", adviceMode));
		}
		return imports;
	}
```


第一步实现原理如下 


### 查找父类型引用

org.springframework.core.ResolvableType#as

```java
	public ResolvableType as(Class<?> type) {
		if (this == NONE) {
			return NONE;
		}
        // 1. 当前类型就是匹配的类型，则返回
		if (ObjectUtils.nullSafeEquals(resolve(), type)) {
			return this;
		}
        // 2. 递归查询当前类型的接口;
		for (ResolvableType interfaceType : getInterfaces()) {
			ResolvableType interfaceAsType = interfaceType.as(type);
			if (interfaceAsType != NONE) {
				return interfaceAsType;
			}
		}
        // 3. 递归检测当前类型的父类;
		return getSuperType().as(type);
	}
```


### 事务配置导入

事务配置选择器通过实现 AdviceModeImportSelector 抽象类, 使得

1. 当通知类型为PROXY时，导入配置类型为 AutoProxyRegistrar 和           ProxyTransactionManagementConfiguration
2. 当通知类型为ASPECTJ时，导入配置类型为 AspectJTransactionManagementConfiguration

```java
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {

	/**
	 * {@inheritDoc}
	 * @return {@link ProxyTransactionManagementConfiguration} or
	 * {@code AspectJTransactionManagementConfiguration} for {@code PROXY} and
	 * {@code ASPECTJ} values of {@link EnableTransactionManagement#mode()}, respectively
	 */
	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] {AutoProxyRegistrar.class.getName(), ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME};
			default:
				return null;
		}
	}

}
```


#### AutoProxyRegistrar

在正确配置下可以为IOC容器注入 InfrastructureAdvisorAutoProxyCreator 
使得容器可以在 AbstractAutoProxyCreator#postProcessBeforeInstantiation 钩子处
为每一个实例化后的对象 尝试创建 一个代理类

```java
public class AutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

	private final Log logger = LogFactory.getLog(getClass());

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		// 1. 遍历导入配置类的所有注解;
		boolean candidateFound = false;
		Set<String> annTypes = importingClassMetadata.getAnnotationTypes();
		for (String annType : annTypes) {
			AnnotationAttributes candidate = AnnotationConfigUtils.attributesFor(importingClassMetadata, annType);
			if (candidate == null) {
				continue;
			}
			// 1.1 筛选代理配置
			Object mode = candidate.get("mode");
			Object proxyTargetClass = candidate.get("proxyTargetClass");
			if (mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() &&
					Boolean.class == proxyTargetClass.getClass()) {
				candidateFound = true;
				// 1.1.1 如果通知模式是代理;
				if (mode == AdviceMode.PROXY) {
					//  1.1.1.1 注册自动代理创建器 InfrastructureAdvisorAutoProxyCreator
					AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
					//  1.1.1.2 如果代理目标类, 则使用CGLIB方式创建;
					if ((Boolean) proxyTargetClass) {
						AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
						return;
					}
				}
			}
		}
		// 2. 找不到代理配置 则打印日志;
		if (!candidateFound && logger.isInfoEnabled()) {
			String name = getClass().getSimpleName();
			logger.info(String.format("%s was imported but no annotations were found " +
					"having both 'mode' and 'proxyTargetClass' attributes of type " +
					"AdviceMode and boolean respectively. This means that auto proxy " +
					"creator registration and configuration may not have occurred as " +
					"intended, and components may not be proxied as expected. Check to " +
					"ensure that %s has been @Import'ed on the same class where these " +
					"annotations are declared; otherwise remove the import of %s " +
					"altogether.", name, name, name));
		}
	}

}

```


#### ProxyTransactionManagementConfiguration

这里定义了 ROLE_INFRASTRUCTURE 角色的切点、通知和Advisor 
并且此 Advisor 可以被 InfrastructureAdvisorAutoProxyCreator 在每个bean创建时扫描到;

由 AnnotationTransactionAttributeSource 可知其创建了支持三种解析起匹配切点 
1. SpringTransactionAnnotationParser
2. JtaTransactionAnnotationParser
3. Ejb3TransactionAnnotationParser

```java
/**
 * {@code @Configuration} class that registers the Spring infrastructure beans
 * necessary to enable proxy-based annotation-driven transaction management.
 *
 * @author Chris Beams
 * @since 3.1
 * @see EnableTransactionManagement
 * @see TransactionManagementConfigurationSelector
 */
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

	/**
	 * 带切点匹配器的通知对象 定义程序运行时的切面;
	 *
	 * 比如如下类
	 * {@link InfrastructureAdvisorAutoProxyCreator}
	 * 将会在每个bean实例话后, 去寻找该方法可以匹配的通知(拦截器);
	 *   {@link AbstractAutoProxyCreator#postProcessAfterInitialization(java.lang.Object, java.lang.String)}
	 *
	 * 现在这里如果发现接收到的方法中包含了事务相关的注解, 将会返回通知对象, 之后代理创建器会创建代理，
	 * 代理在执行的过程中, 会调用拦截器;
	 *
	 * {@link TransactionInterceptor#invoke(org.aopalliance.intercept.MethodInvocation)}
	 *     {@link TransactionAspectSupport#invokeWithinTransaction(java.lang.reflect.Method, java.lang.Class, org.springframework.transaction.interceptor.TransactionAspectSupport.InvocationCallback)}
	 *
	 *
	 */
	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
		BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
		advisor.setTransactionAttributeSource(transactionAttributeSource());
		advisor.setAdvice(transactionInterceptor());
		if (this.enableTx != null) {
			advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
		}
		return advisor;
	}

	/**
	 * 定义AOP事务的切点;
	 */
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}

	/**
	 * 定义AOP事务的通知(拦截器)
	 */
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionInterceptor transactionInterceptor() {
		TransactionInterceptor interceptor = new TransactionInterceptor();
		interceptor.setTransactionAttributeSource(transactionAttributeSource());
		if (this.txManager != null) {
			interceptor.setTransactionManager(this.txManager);
		}
		return interceptor;
	}

}


```


### 总结 

Spring通过 EnableTransactionManagement 使得程序支持AOP功能
1. 通过 InfrastructureAdvisorAutoProxyCreator 和  BeanFactoryTransactionAttributeSourceAdvisor 可以支持为 @Transaction 注解创建事务代理
2. 事务的通知对象通过TransactionInterceptor提供;
