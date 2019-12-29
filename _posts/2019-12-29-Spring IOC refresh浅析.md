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

				// 这里会初始化所有实现了 BeanPostProcessors 接口的类 （ApplicationContextAware 属于 ApplicationContextAwareProcessor，也继承于 BeanPostProcessors 但是这个 processor 会在 prepareBeanFactory 提前注册到 beanFactory 中， LoadTimeWeaverAwareProcessor 同上）
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

                // 初始化后的后续工作
				// Last step: publish corresponding event.
				finishRefresh();

```


####

> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.