---
title: "Spring注解解析"
date: 2020-12-22T21:39:16+08:00
draft: false
---
# Spring 解析注解

### 注解中的属性方法定义

AttributeMethods#isAttributeMethod

```JAVA
	/**
	 * 属性方法的定义是 无参数 有返回值得方法;
	 * @param method
	 * @return
	 */
	private static boolean isAttributeMethod(Method method) {
		return (method.getParameterCount() == 0 && method.getReturnType() != void.class);
	}
```

### 获取注解中的所有属性方法

```JAVA
	private static AttributeMethods compute(Class<? extends Annotation> annotationType) {
		// 1. 收集所有方法
		Method[] methods = annotationType.getDeclaredMethods();
		int size = methods.length;
		// 2. 过滤得到属性方法;
		for (int i = 0; i < methods.length; i++) {
			if (!isAttributeMethod(methods[i])) {
				methods[i] = null;
				size--;
			}
		}
		if (size == 0) {
			return NONE;
		}
		// 3. 排序后输出;
		Arrays.sort(methods, methodComparator);
		Method[] attributeMethods = Arrays.copyOf(methods, size);
		return new AttributeMethods(annotationType, attributeMethods);
	}
```

### 解析别名

```JAVA
	/**
	 * < 目标方法, 别名源方法 >
	 * @return
	 */
	private Map<Method, List<Method>> resolveAliasedForTargets() {
		Map<Method, List<Method>> aliasedBy = new HashMap<>();
		for (int i = 0; i < this.attributes.size(); i++) {
			Method attribute = this.attributes.get(i);
			AliasFor aliasFor = AnnotationsScanner.getDeclaredAnnotation(attribute, AliasFor.class);
			if (aliasFor != null) {
				Method target = resolveAliasTarget(attribute, aliasFor);
				aliasedBy.computeIfAbsent(target, key -> new ArrayList<>()).add(attribute);
			}
		}
		return Collections.unmodifiableMap(aliasedBy);
	}
```
