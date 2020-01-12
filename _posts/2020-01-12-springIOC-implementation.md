---
layout: post
title: Spring IOC实现原理
categories: Blog
description: 本文将介绍基于spring5.1.3的xml配置依赖注入和控制反转的实现原理
keywords: springIOC
---

本文将介绍Spring版本5.1.3 IOC的实现原理及相关知识点。

## Spring IOC 实现原理

#### Bean的加载和注册

首先我们以xml配置，applicationContext作为IOC容器为例来讲解IOC的实现原理

![](/study/images/blog/ioc/ioc1.png)
![](/study/images/blog/ioc/ioc2.png)

将xml配置文件路径作为参数传入ClassPathXmlApplicationContext的构造方法，一路点进refresh()方法。

![](/study/images/blog/ioc/ioc3.png)

可以看到obtainFreshBeanFactory()方法创建了BeanFactory（实现类是DefaultListableBeanFactory），xml配置文件的加载，bean的注册都在其中完成。
点进这个方法进入AbstractRefreshableApplicationContext的loadBeanDefinitions(beanFactory)
![](/study/images/blog/ioc/ioc4.png)

通过beanFactory创建一个XmlBeanDefinitionReader用于读取xml配置，并且将ApplicationContext本身作为ResourceLoader传入XmlBeanDefinitionReader，这样XmlBeanDefinitionReader就具备了获取Resource的能力。
![](/study/images/blog/ioc/ioc5.png)

继续往里，由于测试使用的是ApplicationContext作为ResourceLoader，其本身继承了ResourcePatternResolver，因此会进入下图的逻辑。通过ResourceLoader获取Resource
![](/study/images/blog/ioc/ioc6.png)

继续往里，通过Resource中获取InputStream，进入doLoadBeanDefinitions方法
![](/study/images/blog/ioc/ioc7.png)

doLoadBeanDefinitions中会通过InputStream生成一个Document对象，也就相当于Xml配置文件的抽象表示。随后在registerBeanDefinitions中创建DocumentReader用于解析Xml配置文件。
![](/study/images/blog/ioc/ioc8.png)

继续往里，创建了一个BeanDefinitionParseDelegate作为BeanDefinition解析用的对象
![](/study/images/blog/ioc/ioc9.png)

继续往里，进入parseDefaultElement方法。可以看到delegate对象开始扫描xml配置文件中具体的dom元素，根据不同标签进入对应逻辑
![](/study/images/blog/ioc/ioc10.png)

如果读入的dom元素是<alias>，则会将alias注册在AliasRegistry的缓存中，key是alias，value是bean id
![](/study/images/blog/ioc/ioc11.png)
![](/study/images/blog/ioc/ioc12.png)

如果读到的dom元素是<bean>，则会创建BeanDefinitionHolder对象，对应于<bean>标签里配置内容
![](/study/images/blog/ioc/ioc13.png)

需要注意的是bean id会保存在beanName字段中，但是beanDefinition中只会存有除bean名字外的配置
![](/study/images/blog/ioc/ioc14.png)

继续往里进入registerBeanDefinition方法，bean的id作为key，beanDefinition作为value被存放在DefaultListableBeanFactory的beanDefinitionMap中，同时id还被存入beanDefinitionNames中。至此bean的加载和注册完成。
![](/study/images/blog/ioc/ioc15.png)

#### Bean实例化和依赖注入
说完了Bean的加载和注册，我们来看一下Bean依赖注入的过程。首先回到之前的refresh方法，其中调用了finishBeanFactoryInitialization来进行依赖注入的。我们点进去往里看，最后一行调用了preInstantiateSingletons方法，从字面上也能理解是预先实例化单例bean对象。
![](/study/images/blog/ioc/ioc16.png)

往里看，将之前存在DefaultListableBeanFactory中的beanDefinitionNames拿出来，通过遍历bean id实例化对应的bean，如果bean在配置时指定是通过FactoryBean的形式实例化就走中间逻辑（FactoryBean构建Bean实例会在后面提到），否则就直接进入下方的getBean
![](/study/images/blog/ioc/ioc17.png)

进入getBean→doGetBean方法，首先会从缓存中查看bean是否已经被实例化，getMergedLocalBeanDefinition方法会获取父类bean的definition并合并到子类。然后根据bean scope进入对应的createBean逻辑。如果是singleton bean，在getSingleton方法中会把通过createBean方法创建的bean加入到BeanFactory的singletonObjects中，也就是单例bean的缓存。下次通过bean id来getBean的时候就可以直接从缓存中获取到对应的bean对象了。
![](/study/images/blog/ioc/ioc18.png)

以创建singleton对象为例，进入createBean首先会从缓存中查找是否有bean实现了InstantiationAwareBeanPostProcessor，有的话则调用resolveBeforeInstantiation返回Bean的代理对象，如果没有则进入doCreateBean方法
![](/study/images/blog/ioc/ioc19.png)

doCreateBean方法中，调用createBeanInstance实例化bean
![](/study/images/blog/ioc/ioc20.png)

进入createBeanInstance→instantiateBean→instantiate方法。 可以看到先通过RootBeanDefinition获取bean对象的class类型，再通过反射获取class的构造器，然后进入BeanUtils.instantiateClass
![](/study/images/blog/ioc/ioc21.png)

可以看到bean的实例化最终是通过反射机制实现的。获取bean以后，在instantiateBean方法中，将bean本身作为参数来创建包装类BeanWrapper的对象，BeanWrapperImpl中有内部类BeanPropertyHandler用来执行setter依赖注入。
![](/study/images/blog/ioc/ioc22.png)

回到上方doCreateBean方法中，在这个关键的try{}块当中，populateBean方法实现了bean的setter依赖注入。initializeBean方法中，还对初始化以后的bean进行了额外操作。
![](/study/images/blog/ioc/ioc23.png)

先看populateBean方法。从RootBeanDefinition上获取PropertyValues对象，也就是xml中<property>标签所包含的内容。进入下方applyPropertyValues方法
![](/study/images/blog/ioc/ioc24.png)

在applyPropertyValues方法中，新建了BeanDefinitionValueResolver用于返回需要注入的基本数据类型值或者对象引用，新建一个deepCopy ArrayList用于记录需要进行setter注入的PropertyValue对象。for循环遍历PropertyValue，调用valueResolver.resolveValue获取<property>标签中注入的基本数据类型值或者引用对象。
![](/study/images/blog/ioc/ioc25.png)

进入resolveValue方法，如果需要注入的值是基本数据类型，则返回该值
![](/study/images/blog/ioc/ioc26.png)

如果需要注入的是对象引用，则进入resolveReference方法
![](/study/images/blog/ioc/ioc27.png)

可以看到在resolveReference方法中调用了beanFactory.getBean去获取引用对象，如果该对象还没有实例化，就会在beanFactory中实例化再返回，也就意味着bean的依赖是通过递归寻找来实现的。
![](/study/images/blog/ioc/ioc28.png)

回到applyPropertyValues方法中，将之前处理完的deepCopy ArrayList传入，进行setter注入
![](/study/images/blog/ioc/ioc29.png)

进入setPropertyValues方法，一路debug最终就会进入到BeanWrapperImpl的内部类BeanPropertyHandler的setValue方法中。可以看到最终也是通过反射来调用setter方法实现setter注入的。
![](/study/images/blog/ioc/ioc30.png)

说完了populateBean方法内的逻辑，现在来看一下initializeBean方法。
invokeAwareMethods方法，如果bean实现了BeanNameAware / BeanClassLoaderAware / BeanFactoryAware，就调用这些接口中实现的方法。
applyBeanPostProcessorsBeforeInitialization方法中，如果我们自己实现了BeanPostProcessor，就可以调用我们实现的postProcessBeforeInitialization方法。
invokeInitMethods方法，如果在bean中指定了init-method，则会调用对应方法。
applyBeanPostProcessorAfterInitialization方法中，如果我们自己实现了BeanPostProcessor，就可以调用我们实现的postProcessAfterInitialization方法。
![](/study/images/blog/ioc/ioc31.png)


## IOC用到的设计模式
####工厂模式
Spring使用工厂模式可以通过BeanFactory或ApplicationContext来创建 bean 对象。
两者对比：
BeanFactory ：延迟注入(使用到某个bean的时候才会注入)，相比于ApplicationContext来说会占用更少的内存，程序启动速度更快。
ApplicationContext ：容器启动的时候，一次性创建所有 非懒加载的bean。BeanFactory 仅提供了最基本的依赖注入支持，ApplicationContext是BeanFactory的子接口,除了有BeanFactory的功能还有一些额外功能。

####单例模式
在Spring中bean的默认scope就是singleton。实例化完成的单例bean都会缓存在BeanFactory的ConcurrentHashMap对象singletonObjects中。

####装饰者模式
在Spring中将bean本身作为参数来生成bean的包装类对象BeanWrapperImpl，在不改变原有对象的情况下为对象提供了扩展功能。

####策略模式
在依赖注入的过程中，Spring会调用ApplicationContext 来获取Resource的实例。然而，Resource 接口封装了各种可能的资源类型，包括了：UrlResource，ClassPathResource，FileSystemResource等，Spring需要针对不同的资源采取不同的访问策略。在这里，Spring让ApplicationContext成为了资源访问策略的“决策者”。在资源访问策略的选择上，Spring采用了策略模式。当 Spring 应用需要进行资源访问时，它并不需要直接使用 Resource 实现类，而是调用 ApplicationContext 实例的 getResource() 方法来获得资源，ApplicationContext 将会负责选择 Resource 的实现类，也就是确定具体的资源访问策略，从而将应用程序和具体的资源访问策略分离开来。

