---
layout:     post
title:      Java Spring IOC 源码浅析（refresh 方法）
subtitle:   日常记录
date:       2019-12-29
author:     fyypumpkin
header-img: img/spring-ioc-bg.jpg
catalog: true
tags:
    - Java
    - Spring
    - IOC
---

## 正文

好久没写博客了，最近看了下 Spring IOC 相关的内容，之前一直知道个大概，但是没有仔细的了解过相关的内容，最近有机会，了解了一下相关的内容，
主要研究了下 AbstractApplicationContex 里面的 refresh 相关的代码，这里包含了整个 bean 的创建过程，本文就会针对 refresh 里面的代码解析一遍，主要会解析重点部分，
下面就看下整个 refresh 的流程

### 简单看一下 refresh 大致流程

![](/assets/img/Spring-refresh.png)

只画了比较关键的部分，大致分为 12 个步骤

```java

// Prepare this context for refreshing.
// 启动容器的一些准备，环境设置等
prepareRefresh();

// 这里会生成一个  beanFactory，替换当前的 beanFactory （DefaultListableBeanFactory）
// 同时，配置文件就会解析成一个个 Bean 定义，注册到 BeanFactory 中，当然，这里说的 Bean 还没有初始化，只是配置信息都提取出来了，
// 注册也只是将这些信息都保存到了注册中心(说到底核心是一个 beanName -> beanDefinition 的 map)，
// Tell the subclass to refresh the internal bean factory.
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

// 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，
// 例如常用的 LoadTimeWeaverAwareProcessor 和 ApplicationContextAwareProcessor 这两个 BeanPostProcessors 就是在这里注册的，而不是在 registerBeanPostProcessors 注册的，
// Prepare the bean factory for use in this context.
prepareBeanFactory(beanFactory);

try {

	// 这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
	// 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。
	// 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 定义都加载、注册完成了，但是都还没有初始化
	// 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
	// Allows post-processing of the bean factory in context subclasses.
	postProcessBeanFactory(beanFactory);

	// 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 方法
	// Invoke factory processors registered as beans in the context.
	// 这里会初始化所有实现了 BeanFactoryPostProcessors 接口的类,顺序：
	// 1.BeanDefinitionRegistryPostProcessor （PriorityOrdered）
          // 2.BeanDefinitionRegistryPostProcessor （Ordered）
          // 3. BeanDefinitionRegistryPostProcessor
          // 4.BeanFactoryPostProcessor
	invokeBeanFactoryPostProcessors(beanFactory);

	// 这里会初始化所有实现了 BeanPostProcessors 接口的类
	// （ApplicationContextAware 属于 ApplicationContextAwareProcessor，也继承于 BeanPostProcessors 但是这个 processor 会在 prepareBeanFactory 提前注册到 beanFactory 中， LoadTimeWeaverAwareProcessor 同上）
	// InstantiationAwareBeanPostProcessor 注册会写标识 （这个后续会单独开博客讲一下）
	// Register bean processors that intercept bean creation.
	registerBeanPostProcessors(beanFactory);

          // 初始化国际化资源
	// Initialize message source for this context.
	initMessageSource();

          // 初始化应用事件广播
	// Initialize event multicaster for this context.
	initApplicationEventMulticaster();

          // 模板方法，子类实现(ServletWebServerApplicationContext) 重写了，加了 servlet 容器相关 bean 的初始化
	// Initialize other special beans in specific context subclasses.
	onRefresh();

          // 注册 bean 定义事件监听
	// Check for listener beans and register them.
	registerListeners();

	// 完成剩余的初始化任务，初始化所有非懒加载且未初始化的 bean
	// Instantiate all remaining (non-lazy-init) singletons.
	finishBeanFactoryInitialization(beanFactory);

          // 初始化后的后续
	// Last step: publish corresponding event.
	finishRefresh();

```

上面的流程下面就重要的方法一一详述

#### obtainFreshBeanFactory

refreshBeanFactory 里面主要就是新建了一个 beanFactory，同时加载 bean 定义

```java

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	refreshBeanFactory();
	return getBeanFactory();
}

/**
 * This implementation performs an actual refresh of this context's underlying
 * bean factory, shutting down the previous bean factory (if any) and
 * initializing a fresh bean factory for the next phase of the context's lifecycle.
 * BeanFactory
 * ListableBeanFactory HierarchicalBeanFactory AutowireCapableBeanFactory
 *     |            |   |                                         |
 *     |        ApplicationContext                                |
 *     |                                                          |
 * ConfigurableListableBeanFactory                    AbstractAutowireCapableBeanFactory
 *                              |                      |
 *                             DefaultListableBeanFactory
 */
@Override
protected final void refreshBeanFactory() throws BeansException {
    // 当前有 beanFactory 就销毁
	if (hasBeanFactory()) {
		// 清除 bean
		destroyBeans();
		// 清除 bean 工厂
		closeBeanFactory();
	}
	try {
		// 新建一个 BeanFactory （DefaultListableBeanFactory）
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		// 自定义工厂配置 （允许循环引用，覆盖定义）
		customizeBeanFactory(beanFactory);
		// 加载 bean 定义 （BeanDefinition）（加载 bean 定义会在之后博客中解析）
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	} catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}

```


### prepareBeanFactory

这个方法比较重要的地方在于设置了一个 ApplicationContextAwareProcessor，这个我们平时继承 ApplicationContextAware

```java
// 设置类加载器 （使用加载此类的类加载器）
// Tell the internal bean factory to use the context's class loader etc.
beanFactory.setBeanClassLoader(getClassLoader());
beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

// 添加一个 BeanPostProcessor，这个 processor 比较简单：
// 实现了 Aware 接口的 beans 在初始化的时候，这个 processor 负责回调，
// 这个我们很常用，如我们会为了获取 ApplicationContext 而 implement ApplicationContextAware
// 注意：它不仅仅回调 ApplicationContextAware，
// 还会负责回调 EnvironmentAware、ResourceLoaderAware 等
// Configure the bean factory with context callbacks.
beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

// BeanFactory interface not registered as resolvable type in a plain factory.
// MessageSource registered (and found for autowiring) as a bean.
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
beanFactory.registerResolvableDependency(ResourceLoader.class, this);
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
beanFactory.registerResolvableDependency(ApplicationContext.class, this);

// Register early post-processor for detecting inner beans as ApplicationListeners.
beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

// Detect a LoadTimeWeaver and prepare for weaving, if found.
if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
	beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
	// Set a temporary ClassLoader for type matching.
	beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
}

// Register default environment beans.
if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
	beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
}
if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
	beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
}
if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
	beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
}
```

### invokeBeanFactoryPostProcessors

这一步会初始化实现了 BeanFactoryPostProcessors 接口的类，BeanDefinitionRegistryPostProcessor 这个特殊的接口会优先筛选出来，并且按照实现了 PriorityOrdered， Ordered 和没有实现优先级相关接口的顺序进行 bean 的初始化切调用响应的回调方法，处理完 BeanDefinitionRegistryPostProcessor 相关的类之后，初始化所有实现了 BeanFactoryPostProcessors 的接口，并且按照上面的优先级进行 bean 的初始化，下面看下代码，大部分都会有注释解释

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
  // 调用 BeanFactoryPostProcessor
  // 例如可以通过 BeanDefinitionRegistryPostProcessor 新加自己的 bean 定义（DubboContainer）
  PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

  // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
  // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
  if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
    beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
    beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
  }
}

public static void invokeBeanFactoryPostProcessors(
    ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

  // 所有实现了 BeanFactoryPostProcessor （包括 BeanDefinitionRegistryPostProcessor） 并且已经执行过回调方法的 beanName
  // Invoke BeanDefinitionRegistryPostProcessors first, if any.
  Set<String> processedBeans = new HashSet<>();

  // 判断beanFactory是否为BeanDefinitionRegistry，在这里普通的beanFactory是DefaultListableBeanFactory,而DefaultListableBeanFactory实现了BeanDefinitionRegistry接口，因此这边为true
  // 这里会调用实现初始化好的 BeanFactoryPostProcessor 并调用回调
  if (beanFactory instanceof BeanDefinitionRegistry) {
    BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
    List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
    List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

    for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
      if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
        BeanDefinitionRegistryPostProcessor registryProcessor =
            (BeanDefinitionRegistryPostProcessor) postProcessor;
        registryProcessor.postProcessBeanDefinitionRegistry(registry);
        registryProcessors.add(registryProcessor);
      }
      else {
        regularPostProcessors.add(postProcessor);
      }
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    // Separate between BeanDefinitionRegistryPostProcessors that implement
    // PriorityOrdered, Ordered, and the rest.
    List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

    // 先扫描实现了 BeanDefinitionRegistryPostProcessor 并且是 PriorityOrdered 的 beanName
    // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
    String[] postProcessorNames =
        beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
    for (String ppName : postProcessorNames) {
      if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        processedBeans.add(ppName);
      }
    }
    sortPostProcessors(currentRegistryProcessors, beanFactory);
    registryProcessors.addAll(currentRegistryProcessors);
    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
    currentRegistryProcessors.clear();

    // 然后先扫描实现了 BeanDefinitionRegistryPostProcessor 并且是 Ordered 的 beanName
    // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
    postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
    for (String ppName : postProcessorNames) {
      if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        processedBeans.add(ppName);
      }
    }
    sortPostProcessors(currentRegistryProcessors, beanFactory);
    registryProcessors.addAll(currentRegistryProcessors);
    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
    currentRegistryProcessors.clear();

    // 再循环获取实现了 BeanDefinitionRegistryPostProcessor， 如果在当前的 set 中没有，就调用 getBean 方法获取 bean（
    // 这里实际上我认为是会初始化整个 bean 的）getBean 方法在后续的 finishBeanFactoryInitialization 中解析
    // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
    boolean reiterate = true;
    while (reiterate) {
      reiterate = false;
      postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
        if (!processedBeans.contains(ppName)) {
          currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
          processedBeans.add(ppName);
          reiterate = true;
        }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      // 调用 postProcessBeanDefinitionRegistry 方法
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();
    }

    // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
    invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
  }

  else {
    // Invoke factory processors registered with the context instance.
    invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
  }

  // 最后扫描所有实现了 BeanFactoryPostProcessor 的 bean，和上面操作类似
  // Do not initialize FactoryBeans here: We need to leave all regular beans
  // uninitialized to let the bean factory post-processors apply to them!
  String[] postProcessorNames =
      beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

  // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
  // Ordered, and the rest.
  List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
  List<String> orderedPostProcessorNames = new ArrayList<>();
  List<String> nonOrderedPostProcessorNames = new ArrayList<>();
  for (String ppName : postProcessorNames) {
    if (processedBeans.contains(ppName)) {
      // skip - already processed in first phase above
    }
    else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
      priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
    }
    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
      orderedPostProcessorNames.add(ppName);
    }
    else {
      nonOrderedPostProcessorNames.add(ppName);
    }
  }

  // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
  sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
  invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

  // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
  List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
  for (String postProcessorName : orderedPostProcessorNames) {
    orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  }
  sortPostProcessors(orderedPostProcessors, beanFactory);
  invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

  // Finally, invoke all other BeanFactoryPostProcessors.
  List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
  for (String postProcessorName : nonOrderedPostProcessorNames) {
    nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  }

  invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

  // Clear cached merged bean definitions since the post-processors might have
  // modified the original metadata, e.g. replacing placeholders in values...
  beanFactory.clearMetadataCache();
}

```

### registerBeanPostProcessors

上面的 BeanFactoryPostProcessors 处理完了之后，就要处理我们经常使用的 BeanPostProcessors 接口的 bean，这里的处理和上面的 BeanFactoryPostProcessor 很像，会将所有实现了 BeanPostProcessor 的接口按照优先级依次初始化，并且注册到当前 beanFactory 中，后续其他 bean 初始化会循环调用这些实现了 BeanPostProcessor 的 bean 的回调方法 （实现了 BeanPostProcess 的类或者是更早处理的类不能懒加载），同时，细看的人会发现，beanPostProcessors 存放的地方是一个 CopyOnWriteArrayList，对于 beanPostProcessors 来说，使用 CopyOnWriteArrayList 是一个很好的选择，因为这个只有在初始化的时候才会频繁写，后续都是读操作。

```java

protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}

public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		// 获取所有实现了 BeanPostProcessor 的 bean 信息
		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		// 计算数量
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// 注册所有实现了 PriorityOrdered 的 bean
		// First, register the BeanPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// 注册所有实现了 Ordered 的 bean
		// Next, register the BeanPostProcessors that implement Ordered.
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// 注册所有常规的 bean
		// Now, register all regular BeanPostProcessors.
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// 注册所有内部的 bean （实现了 MergedBeanDefinitionPostProcessor 的bean）
		// Finally, re-register all internal BeanPostProcessors.
		sortPostProcessors(internalPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}

```

### finishBeanFactoryInitialization

finishBeanFactoryInitialization 方法，这个方法会初始化所有非懒加载的 bean，这个 bean 上层方法比较简单，实际上，上面的方法对于 bean 的初始化都是委托给了 beanFactory 的 getBean 方法，finishBeanFactoryInitialization 会把所有的 bean 定义进行循环初始化， getBean 方法比较大，在后面单独讲。

```java

protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// 一个数据转换的 bean 可以暂时不理会
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// 初始化所有剩下的且为非懒加载的 bean （Ps.之前直接或者间接实现了相应 BeanPostProcessor 和 BeanFactoryPostProcessor 的 bean 已经被初始化完成了）
		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}

  @Override
  	public void preInstantiateSingletons() throws BeansException {
  		if (logger.isTraceEnabled()) {
  			logger.trace("Pre-instantiating singletons in " + this);
  		}

  		// 获取所有 bean 定义信息
  		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
  		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
  		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

  		// 对这些信息都去获取一遍他的 bean 对象
  		// Trigger initialization of all non-lazy singleton beans...
  		for (String beanName : beanNames) {
  			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
  			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
  				// 如果是 FactoryBean，就在 beanName 前面加上 & 符号 （这里后续也会开个坑讲一讲，一般只有在复杂 bean 初始化才会用到，比如 SqlSessionFactoryBean， Dubbo 中的 ReferenceBean）
  				if (isFactoryBean(beanName)) {
  					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
  					if (bean instanceof FactoryBean) {
  						final FactoryBean<?> factory = (FactoryBean<?>) bean;
  						boolean isEagerInit;
  						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
  							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
  											((SmartFactoryBean<?>) factory)::isEagerInit,
  									getAccessControlContext());
  						}
  						else {
  							isEagerInit = (factory instanceof SmartFactoryBean &&
  									((SmartFactoryBean<?>) factory).isEagerInit());
  						}
  						if (isEagerInit) {
  							getBean(beanName);
  						}
  					}
  				}
  				else {
  					// 获取 bean
  					getBean(beanName);
  				}
  			}
  		}

  		// Trigger post-initialization callback for all applicable beans...
  		for (String beanName : beanNames) {
  			Object singletonInstance = getSingleton(beanName);
  			if (singletonInstance instanceof SmartInitializingSingleton) {
  				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
  				if (System.getSecurityManager() != null) {
  					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
  						smartSingleton.afterSingletonsInstantiated();
  						return null;
  					}, getAccessControlContext());
  				}
  				else {
  					smartSingleton.afterSingletonsInstantiated();
  				}
  			}
  		}
}
```

### getBean

getBean 方法是获取 bean 信息的入口，若没有相应的示例则会创建，可见整个 bean 的创建过程都包在了 getBean 方法中，同时，在 getBean 方法中也有比较重要的如何处理循环引用，什么是循环引用？A 服务依赖了 B 服务， B 服务又依赖了 A 服务，我们知道，Spring 非构造器注入才支持处理这种循环引用，仔细想一下，如果是构造器注入，肯定没有办法处理这种循环应用的问题，那么 Spring 非构造器是如何处理的呢？

这里就会涉及到 Spring 中的 bean 三级缓存，即 singletonObjects，earlySingletonObjects 和 singletonFactories，顾名思义，singletonObjects 就是完成了实例化和初始化的 bean，earlySingletonObjects 是早期的 bean 信息，我认为可以理解为完成了实例化，但还没有初始化的 bean，singletonFactories 使用与获取早期 bean 的工厂，这里有一个问题，为什么需要 3 级缓存，2 级缓存理论上也是可以的name为什么要做成 3 级缓存呢? 这里主要是方便对早期 bean 进行干预处理，使用 bean 工厂 的形式获取早期 bean 时，会调用实现了 SmartInstantiationAwareBeanPostProcessor 的回调方法（如果实现了）。

一般情况下，调用 3 级缓存获取到 earlyBean 后，就会将三级缓存去除，并将 earlyBean 加入 二级缓存，bean 构建完成后，会将二级缓存去除，并将初始化完成的 bean 放入一级缓存 （我们这里讨论的都是单例模式下的）

说完缓存，下面就来讲下 Spring 到底如何处理循环依赖的，假设我有两个 Service，这两个 Service 相互依赖，分别为 AService 和 BService，在 Spring 初始化 Bean 的时候，我们让 AService 先初始化，AService 进入了 getBean.getSingleton(String beanName) 方法，getSingleton(String beanName) 方法会向刚才的三级缓存依次查询，先查询一级缓存，一级缓存没有就去查询 二级缓存，二级缓存没有就去查三级缓存（三级缓存查到了就会将三级缓存删除，并加入二级缓存），当然，我们这里 AService 并不会存在任何一级缓存中，于是经过一系列的操作就进入另一个 getSingleton(String beanName, ObjectFactory<?> singletonFactory) 方法中，进入该方法会先设置一个当前 bean 正在创建中的标识（加入 singletonsCurrentlyInCreation），然后调用 singletonFactory 的 getObject 方法，这个方法就是上层设置的 createBean 方法，进入 createBean 方法中，首先会实例化 bean，注意是实例化，属性还都没有被设置，同时，会将这个实例化出来的 earlyBean 加入到三级缓存中，然后进入 bean 的属性装配，也就是依赖注入，我们这里发现 AService 依赖了 BService，于是通过 beanFactory.getBean(BServiceName) 方法来获取 BService（递归了），这里 BService 同样经历了上面的操作后，加入三级缓存，并且进行依赖注入，通过 getBean(AServiceName) 获取 AService，这个时候，会从三级缓存中获取到 Aservice，并将 AService 放入到二级缓存中，并将这个 earlyBean 的引用设置到 BService 中，BService 完成了装备后，返回装配好的 bean， 在 getSingleton(String beanName, ObjectFactory<?> singletonFactory)  的 finally 块中，会将上面设置的 singletonsCurrentlyInCreation 标识，二级缓存以及三级缓存都去除，并将 bean 加入一级缓存，然后返回回装配 AService 的流程中，AService 成功拿到了 BService 的引用，也完成了装配过程，也将设置的 singletonsCurrentlyInCreation 标识，二级缓存以及三级缓存都去除，然后加入一级缓存，至此整个处理过程就完成了，下面罗列下各个阶段，三个缓存中的内容：

- AService 实例化后，将 earlyBean 加入三级缓存 singletonFactories 中，供其他 bean 使用，此时三级缓存中有 AService
- AService 通过依赖注入 BService，从缓存中获取 BService，发现缓存中没有，开始创建 BService 实例，此时三级缓存中有 AService
- BService 实例化后，也加入三级缓存中，此时三级缓存中有 AService 和 BService
- BService 在进行依赖注入时，发现有 AService 引用，此时，创建 AService 时，会先从缓存中获取AService(先从一级缓存中取，没有取到在从二级缓存中取，还没有取到，再从三级缓存中取出)，三级缓存取出后，会清除三级缓存中的 AService，将其将其加入二级缓存 earlySingletonObjects 中，并返回给 BService 供其使用，此时三级缓存有 BService，二级缓存中有 AService
- BService 在完成依赖注入，进行初始化，完成后，会加入一级缓存，这时会清除三级缓存中的 BService，此时三级缓存为空，二级缓存中有 AService，一级缓存中有 BService
- BService 初始化后注入 AService 中，AService 继续进行初始化，然后通过 getSingleton 方法获取二级缓存，赋值给 exposedObject，最后将其加入一级缓存，清除二级缓存 AService，此时二三级缓存都没数据，一级缓存有 AService 和 BService

> getBean 的源码

```java

protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		// 对 name 做转换 （我们可以对 bean 设置多个别名）
		final String beanName = transformedBeanName(name);
		Object bean;

		// 从容器中获取 bean 对象 （这里会涉及到提前暴露的引用，后续开坑，目前拿到的信息认为 null）
		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// 处理 bean 内的依赖问题，对依赖的 bean 依次初始化 （这里不是正常的 bean 的依赖， 可以不理会）
				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						// 会将相关被依赖的 bean 注册到一个容器中
						registerDependentBean(dep, beanName);
						try {
							// 初始化 bean
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 下面会对 bean 进行创建和初始化，主要关注 createBean 方法
				// 单例形式的
				// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				// 原型模式的
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				// Scope 自实现
				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
}

// 获取实例
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  // 一级缓存
		Object singletonObject = this.singletonObjects.get(beanName);
    // 一级缓存为空，并且当前的类在创建中，就从二级缓存获取
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
        // 如果二级缓存也是空的，就从三级缓存获取，取到，就把二级缓存清除
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
}

public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		synchronized (this.singletonObjects) {
			// 一级缓存未获取到
			Object singletonObject = this.singletonObjects.get(beanName);
			// 空的说明需要创建，使用传进来的 ObjectFactory 创建
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}

				// 将当前的 beanName 加入 singletonsCurrentlyInCreation 容器中
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
					// 实例化初始化一个 bean （这里在getBean 总就是调用了 createBean）
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime ->
					// if yes, proceed with it since the exception indicates that state.
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
					// 这里会去除之前提前暴露出来的 bean （earlySingletonObjects），并将 bean 加入一级缓存，同时去除二三级缓存
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
}

```

> createBean 的源码

```java

@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {

  if (logger.isTraceEnabled()) {
    logger.trace("Creating instance of bean '" + beanName + "'");
  }
  RootBeanDefinition mbdToUse = mbd;

  // 判断当前要创建的Bean是否可以实例化，是否可以通过类装载器来载入
  // Make sure bean class is actually resolved at this point, and
  // clone the bean definition in case of a dynamically resolved Class
  // which cannot be stored in the shared merged bean definition.
  Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
  if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
    mbdToUse = new RootBeanDefinition(mbd);
    mbdToUse.setBeanClass(resolvedClass);
  }

  // Prepare method overrides.
  try {
    mbdToUse.prepareMethodOverrides();
  }
  catch (BeanDefinitionValidationException ex) {
    throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
        beanName, "Validation of method overrides failed", ex);
  }

  try {
    // 这里可以有一个机会修改 bean 信息（例如返回代理对象），需要实现 InstantiationAwareBeanPostProcessor 接口
    // Instantiation 和 Initialization , 实例化和初始化, 先实例化,然后初始化
    // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
      return bean;
    }
  }
  catch (Throwable ex) {
    throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
        "BeanPostProcessor before instantiation of bean failed", ex);
  }

  try {
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    if (logger.isTraceEnabled()) {
      logger.trace("Finished creating instance of bean '" + beanName + "'");
    }
    return beanInstance;
  }
  catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
    // A previously detected exception with proper bean creation context already,
    // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
    throw ex;
  }
  catch (Throwable ex) {
    throw new BeanCreationException(
        mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
  }
}

// 这里解释一下，BeanPostProcessor 作用域对象的初始化，而且 InstantiationAwareBeanPostProcessor 作用于对象的实例化
@Nullable
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
  Object bean = null;
  if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
    // Make sure bean class is actually resolved at this point.
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      Class<?> targetType = determineTargetType(beanName, mbd);
      if (targetType != null) {
        // 实例化前
        bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
        if (bean != null) {
          // 初始化后（bean 不为空，整个 getBean 就返回了）
          bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        }
      }
    }
    mbd.beforeInstantiationResolved = (bean != null);
  }
  return bean;
}

protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {

  // Instantiate the bean.
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
  }
  if (instanceWrapper == null) {
    // 初始化 bean （会涉及到构造器注入）
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }
  final Object bean = instanceWrapper.getWrappedInstance();
  Class<?> beanType = instanceWrapper.getWrappedClass();
  if (beanType != NullBean.class) {
    mbd.resolvedTargetType = beanType;
  }

  // Allow post-processors to modify the merged bean definition.
  synchronized (mbd.postProcessingLock) {
    if (!mbd.postProcessed) {
      try {
        applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
      }
      catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Post-processing of merged bean definition failed", ex);
      }
      mbd.postProcessed = true;
    }
  }

  // 提前曝光bean（利用缓存），用于支持循环bean的循环依赖
  // 在实例化Bean时，会将beanName存入singletonsCurrentlyInCreation集合中，当发现重复时，即说明有循环依赖，抛出异常
  // Eagerly cache singletons to be able to resolve circular references
  // even when triggered by lifecycle interfaces like BeanFactoryAware.
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
      isSingletonCurrentlyInCreation(beanName));
  if (earlySingletonExposure) {
    if (logger.isTraceEnabled()) {
      logger.trace("Eagerly caching bean '" + beanName +
          "' to allow for resolving potential circular references");
    }

    // 将 bean 加入三级缓存 (这里使用三级缓存，可以方便对早期 bean 干预处理， 实现 SmartInstantiationAwareBeanPostProcessor)
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }

  // Initialize the bean instance.
  Object exposedObject = bean;
  try {
    // 依赖注入
    populateBean(beanName, mbd, instanceWrapper);
    // 初始化 bean
    exposedObject = initializeBean(beanName, exposedObject, mbd);
  }
  catch (Throwable ex) {
    if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
      throw (BeanCreationException) ex;
    }
    else {
      throw new BeanCreationException(
          mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
    }
  }

  if (earlySingletonExposure) {
    Object earlySingletonReference = getSingleton(beanName, false);
    if (earlySingletonReference != null) {
      if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
      }
      else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
        String[] dependentBeans = getDependentBeans(beanName);
        Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
        for (String dependentBean : dependentBeans) {
          if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
            actualDependentBeans.add(dependentBean);
          }
        }
        if (!actualDependentBeans.isEmpty()) {
          throw new BeanCurrentlyInCreationException(beanName,
              "Bean with name '" + beanName + "' has been injected into other beans [" +
              StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
              "] in its raw version as part of a circular reference, but has eventually been " +
              "wrapped. This means that said other beans do not use the final version of the " +
              "bean. This is often the result of over-eager type matching - consider using " +
              "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
        }
      }
    }
  }

  // Register bean as disposable.
  try {
    registerDisposableBeanIfNecessary(beanName, bean, mbd);
  }
  catch (BeanDefinitionValidationException ex) {
    throw new BeanCreationException(
        mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
  }

  return exposedObject;
}

```

> populateBean

这里面会对当前依赖的 bean 调用 getBean 获取依赖的 bean 来进行注入

```java

@SuppressWarnings("deprecation")  // for postProcessPropertyValues
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		if (bw == null) {
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		// 调用实现了 InstantiationAwareBeanPostProcessor 的 after 方法（实例化后方法）
		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						return;
					}
				}
			}
		}

		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

		int resolvedAutowireMode = mbd.getResolvedAutowireMode();
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
			// Add property values based on autowire by name if applicable.
			// 根据名称注入
			if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}
			// Add property values based on autowire by type if applicable.
			// 根据类型注入
			if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}
			pvs = newPvs;
		}

		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

		PropertyDescriptor[] filteredPds = null;
		if (hasInstAwareBpps) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						if (filteredPds == null) {
							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
						}
						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvsToUse == null) {
							return;
						}
					}
					pvs = pvsToUse;
				}
			}
		}
		if (needsDepCheck) {
			if (filteredPds == null) {
				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			}
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}

		if (pvs != null) {
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
}

```

> initializeBean

对 bean 进行初始化，调用初始化方法以及 initializingBean 接口相关回调，还有 BeanPostProcessor 的相关回调

```java

protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
  if (System.getSecurityManager() != null) {
    AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
      invokeAwareMethods(beanName, bean);
      return null;
    }, getAccessControlContext());
  }
  else {
    // 调用 三个 Aware 接口回调 （BeanNameAware， BeanFactoryAware， BeanClassLoaderAware）
    invokeAwareMethods(beanName, bean);
  }

  Object wrappedBean = bean;
  if (mbd == null || !mbd.isSynthetic()) {
    // BeanPostProcessor 的初始化前回调
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
  }

  try {
    // 调用 init-method
    invokeInitMethods(beanName, wrappedBean, mbd);
  }
  catch (Throwable ex) {
    throw new BeanCreationException(
        (mbd != null ? mbd.getResourceDescription() : null),
        beanName, "Invocation of init method failed", ex);
  }
  if (mbd == null || !mbd.isSynthetic()) {
    // BeanPostProcessor 的初始化后回调
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
  }

  return wrappedBean;
}

```

上面的流程还是比较繁琐的，可以结合博客开头的图来帮助理解，整个过程比较关键的部分就是对一些接口注册调用（invokeBeanFactoryPostProcessors，registerBeanPostProcessor） 以及对 bean 的实例化初始化 （getBean）方法的理解，以及 Spring 处理循环依赖，本博客还是以源码分析为主的，可能看起来会比较累，理解了主要过程就会好很多（Ps.我认为大多时候开始学习不需要太过于钻小细节的牛角尖，先理解宏观的过程，后续整整碰到问题的时候或者应用到的时候，再来研究会轻松很多）


####


> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.
