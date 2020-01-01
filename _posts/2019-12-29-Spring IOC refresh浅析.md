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

getBean 方法是获取 bean 信息的入口，若没有相应的示例则会创建，可见整个 bean 的创建过程都包在了 getBean 方法中，同时，在 getBean 方法中也有比较重要的如何处理循环引用，什么是循环引用？A 服务依赖了 B 服务， B 服务又依赖了 A 服务，我们知道，Spring 非构造器注入才支持处理这种循环引用，仔细想一下，如果是构造器注入，肯定没有办法处理这种循环应用的问题，那么 Spring 非构造器是如何处理的呢？这里就会涉及到 Spring 中的 bean 三级缓存，即 singletonObjects，earlySingletonObjects 和 singletonFactories，顾名思义，singletonObjects 就是完成了实例化和初始化的 bean，earlySingletonObjects 是早期的 bean 信息，我认为可以理解为完成了实例化，但还没有初始化的 bean，singletonFactories 使用与获取早期 bean 的工厂

```

####


> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.
