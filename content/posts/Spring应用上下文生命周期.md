---
title: "Spring应用上下文生命周期"
date: 2021-01-13 22:45:43.435
draft: false
type: "post"
showTableOfContents: true
tags: ["Spring","源码"]
---

# Spring应用上下文生命周期

# 1. 前言

本文基于Spring-framework.5.2.2.RELEASE版本进行论述，参考了小马哥在极客时间的课程(文章末尾有对应二维码)。

# 2. Spring应用上下文刷新阶段

刷新阶段主要是围绕 `AbstractApplicationContext#refresh()`进行讨论，这也是所有的应用上下文的抽象父类，可以先简单看一下整个方法的代码，条理鲜明：

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

## 2.1. Spring上下文刷新准备阶段

这一阶段对应 `#prepareRefresh`方法，有以下职责：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled.png](https://img.masaiqi.com/20210113224408.png)

这个要注意的是 `Environment`**(可以理解为应用整体运行环境，用来管理配置信息)**这个类的创建时间点，过一下代码：

```java
protected void prepareRefresh() {
		......

		// Initialize any placeholder property sources in the context environment.
		initPropertySources();

		// Validate that all properties marked as required are resolvable:
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties();

		......
	}
```

看起来是在 `#getEnvironment`的时候进行创建，但实际上，各个子类上下文环境可能会重写 `#initPropertySources`方法，最终会提前调用 `#getEnvironment`方法，比如`StaticWebApplicationContext#initPropertySources`方法：

```java
protected void initPropertySources() {
      WebApplicationContextUtils.initServletPropertySources(this.getEnvironment().getPropertySources(), this.servletContext, this.servletConfig);
}
```

**因此，Environment类可能是在`#initPropertySources`的时候就调用`#getEnvironment`进行初始化，也可能在`getEnvironment().validateRequiredProperties()`才进行初始化。**

除此以外，“初始化早期Spring事件集合”，这个只是代表提供了早期Spring事件的容器，而非填充内容：

```java
// Allow for the collection of early ApplicationEvents,
// to be published once the multicaster is available...
this.earlyApplicationEvents = new LinkedHashSet<>();
```

## 2.2. BeanFactory创建阶段

这一阶段对应 `#obtainFreshBeanFactory`方法，有以下职责：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%201.png](https://img.masaiqi.com/20210113224409.png)

这一阶段特别注意 `#loadBeanDefinitions`方法，这个方法是委派给子类去实现的，抽象父类并没有具体实现，以`AbstractXmlApplicationContext`为例：

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
}
```

总的来说就是初始化各种读取器，在xml中对应 `XmlBeanDefinitionReader`，然后读取对应配置的beanDefinition信息,再通过顶层的父类`DefaultListableBeanFactory#registerBeanDefinition()` 提供能力，完成的BeanDefinition注册，**这里的“注册”意思是将配置信息转化为BeanDefinition并放到合适的容器中。**

我们Debug一下看一下方法栈：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%202.png](https://img.masaiqi.com/20210113224410.png)

**这个流程也是Bean生命周期开始的地方**，可以看我的另一篇文章[SpringBean的生命周期一图流详细说明](https://www.masaiqi.com/archives/springbean%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E4%B8%80%E5%9B%BE%E6%B5%81%E8%AF%A6%E7%BB%86%E8%AF%B4%E6%98%8E)

## 2.3. BeanFactory准备阶段

这一阶段对应 `#prepareBeanFactory`方法，有以下职责：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%203.png](https://img.masaiqi.com/20210113224411.png)

这里的`ApplicationListenerDetector`是主要负责`ApplicationListener`类的一些生命周期处理的内部`BeanPostProcessor`

## 2.4. BeanFactory后置处理阶段

这一阶段对应`#postProcessBeanFactory`方法和`#invokeBeanFactoryPostProcessors`方法，有以下职责：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%204.png](https://img.masaiqi.com/20210113224412.png)

`#postProcessBeanFactory`这个方法是委派给子类去实现的，允许子类添加一些`BeanPostProcessor`进BeanFactory中。

`#invokeBeanFactoryPostProcessors`主要执行一些`BeanFactoryPostProcessor`

## 2.5. BeanFactory注册BeanPostProcessor阶段

这一阶段对应`#registerBeanPostProcessors`方法：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%205.png](https://img.masaiqi.com/20210113224413.png)

这个阶段主要是负责注册其他的BeanPostProcessor，将它们添加到BeanFactory中

## 2.6. 初始化MessageSource

这一阶段主要对应`#initMessageSource`方法，用来初始化Spring国际化所需要的一些内建Bean。

## 2.7. 初始化ApplicationEventMulticaster

这一阶段主要对应`#initApplicationEventMulticaster` 方法，用来初始化Spring事件广播器，用于实现Spring事件机制。

## 2.8. Spring应用上下文刷新阶段

这一阶段对应`#onRefresh`方法：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%206.png](https://img.masaiqi.com/20210113224414.png)

这个方法主要是委派给子类去实现，就像上面图中的部分子类，各个容器环境等会有不一样的实现，各个容器或环境可能有一些启动时需要准备的事情，就放在这个方法里去做。

## 2.9. Spring事件监听器注册阶段

这一阶段对应`#registerListeners`方法：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%207.png](https://img.masaiqi.com/20210113224415.png)

这个方法主要是将上文中的事件监听器`ApplicationListener`注册到之前已经初始化了的事件广播器`ApplicationEventMulticaster`中，并发布一些早期事件(比如通过之前的`BeanFactoryPostProcessor`注册到容器中的事件)

## 2.10. BeanFactory初始化完成阶段

这一阶段主要对应`#finishBeanFactoryInitialization`方法：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%208.png](https://img.masaiqi.com/20210113224416.png)

主要负责初始化一些非延迟的单例Bean，通俗来说就是他最终调用了`AbstractBeanFactory#getBean`方法，也就是一个单例Bean实例化的起点，这个也可以参考我的另一篇文章[SpringBean的生命周期一图流详细说明](https://www.masaiqi.com/archives/springbean%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E4%B8%80%E5%9B%BE%E6%B5%81%E8%AF%A6%E7%BB%86%E8%AF%B4%E6%98%8E)

## 2.11. Spring应用上下文刷新完成阶段

这一阶段对应`#finishRefresh`方法：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%209.png](https://img.masaiqi.com/20210113224417.png)

主要任务还是发布一些上下文完成阶段的事件。

此外，这个阶段有个方法可以注意下：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%2010.png](https://img.masaiqi.com/20210113224418.png)

# 3. Spring应用上下文启动阶段

这个阶段对应`#start`方法：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%2011.png](https://img.masaiqi.com/20210113224419.png)

这个方法非必须，默认也不会去调，**用来处理Lifecycle接口的一些实现:**

```java
public void start() {
		getLifecycleProcessor().start();
		publishEvent(new ContextStartedEvent(this));
}
```

他会调用到：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%2012.png](https://img.masaiqi.com/20210113224420.png)

这个方法在上个阶段也调用到了，但是入参不一样。

# 4. Spring应用上下文停止阶段

这一阶段对应`#stop`方法：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%2013.png](https://img.masaiqi.com/20210113224421.png)

这个方法非必须，默认也不会去调，**用来处理Lifecycle接口的一些实现：**

```java
public void stop() {
		getLifecycleProcessor().stop();
		publishEvent(new ContextStoppedEvent(this));
}
```

# 5. Spring应用上下文关闭阶段

这一阶段对应`#close`方法：

![Spring%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%2073947018b34142d5a2ffffab0e7eef5f/Untitled%2014.png](https://img.masaiqi.com/20210113224422.png)

这个阶段主要是负责处理Spring容器关闭的一些对应操作。

# 6. 参考

![geek.png](https://img.masaiqi.com/20201020123554.png)