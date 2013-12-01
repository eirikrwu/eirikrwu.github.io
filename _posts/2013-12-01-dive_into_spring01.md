---
layout: post
title: "spring源码阅读 01"
---
{% include JB/setup %}

spring是目前java开发中最重要的开源框架之一，是大型项目中依赖管理事实上的行业标准。目前spring本身也已经进化成了一个相对庞大的项目，本文从spring中最常用的入口开始抽丝剥茧，逐步阅读spring源码。

 - 本文适用于spring 3.2.6
 - 不知道什么是spring的同学请移步[spring官方文档](http://docs.spring.io/spring/docs/3.2.6.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#beans)并至少阅读5.1和5.2

<!--more-->

### 阅读环境
spring代码目前托管在[github](https://github.com/spring-projects/spring-framework)上，并且用的是[gradle](http://www.gradle.org/)做项目的构建管理（这个组合非常nice，感觉gradle比ant或者maven都要好用的多）。

强烈建议使用Intellij IDEA阅读spring的代码，它的UML类图自动生成功能非常好用（请参考本文中的类图）。clone spring的git repository以后参考`import-into-idea.md`即可将工程导入Intellij（或者直接使用Intellij导入build.gradle文件应该也可以，Intellij的JetGradle插件支持自动导入gradle项目）。

### 项目结构

导入后可以看到spring代码被整理成若干模块（spring库的发布也是按照这些模块来的），本文涉及到的模块主要为spring-core，spring-bean以及spring-context，这些也是spring中最低层的模块，他们之间的依赖关系如下图所示：

[![XmlBeanFactory类图]({{ site.url }}/assets/dive_into_spring_01/content_dependency.png)]({{ site.url }}/assets/dive_into_spring_01/content_dependency.png)

可以看到spring-core基本上是大家都依赖的，其中主要是一些大家都需要的公共定义和实现等等，spring-bean中实现的是BeanFactory，spring-context在spring-bean之上实现了ApplicationContext。

### 两个接口

spring中最核心的两个接口（其实也可以认为是一个接口）：

 - `org.springframework.context.ApplicationContext` 阅读spring代码最适合的起点应该是这个接口，该接口是外部应用的代码调用spring时最可能的入口，例如，可以根据XML配置文件获得该接口的实例，然后从中得到各种bean的实例等等。
 - `org.springframework.beans.factory.BeanFactory` 简单来说spring本身就是一个高度可配置的[工厂模式](http://en.wikipedia.org/wiki/Factory_method_pattern)，BeanFactory定义了这个工厂的接口。或者用spring的话来说，BeanFactory定义了bean container（容器），这也意味着BeanFactory不仅仅是个工厂（还是个容器），它不仅仅负责创建bean实例，还负责保存bean的配置信息等功能。
 

其实这两个接口也可以被看成是一个接口，因为ApplicationContext实际上是继承BeanFactory的，如下图所示：

[![XmlBeanFactory类图]({{ site.url }}/assets/dive_into_spring_01/application_context_uml.png)]({{ site.url }}/assets/dive_into_spring_01/application_context_uml.png)

可以看出spring中接口的命名中包含很多信息量，比如从图中可以很容易看出，ApplicationContext就是一个Listable/Hierarchical的BeanFactory，并在其上加入了更多的功能，以更好的支持应用。这些功能一部分由ApplcationContext继承的其他接口定义，另一部分直接由它自身包含的方法定义。 

### BeanFactory

spring的最核心功能——管理和生产bean，实际上是由BeanFactory以及它的一系列子类完成的。

由于ApplicationContext也继承了BeanFactory，因此ApplicationContext的实现自然也是BeanFactory的实现，这里暂时不考虑所有这些类。除掉它们之外，BeanFactory中重要的实现只有两个：

 - `org.springframework.beans.factory.support.StaticListableBeanFactory`
 - `org.springframework.beans.factory.support.DefaultListableBeanFactory`

其中StaticListableBeanFactory不太常用，它是一个非常简单的实现，仅仅支持通过代码向其中注册已经存在的单例bean。因此最重要的是DefaultListableBeanFactory，它的类继承关系如下图所示：

[![XmlBeanFactory类图]({{ site.url }}/assets/dive_into_spring_01/xmlbeanfactory_uml.png)]({{ site.url }}/assets/dive_into_spring_01/xmlbeanfactory_uml.png)

可以看到，DefaultListableBeanFactory实现了BeanFactory的三个子接口（BeanFactory也只有这三个直接子接口），这三个子接口分别定义了：

 - `ListableBeanFactory` 可枚举实例的BeanFactory，对ListableBeanFactory中管理的所有Bean实例可以进行遍历，而不是仅仅可以根据名字查找。
 - `HierarchicalBeanFactory` 层次BeanFactory，多个HierarchicalBeanFactory可以构成树状结构。
 - `AutowireCapableBeanFactory` 支持autowire的BeanFactory，在AutowireCapableBeanFactory中生成bean时可以进行autowire，即在一定程度上无需配置就可自动解析出bean之间的依赖关系

spring中接口的定义是很清晰的，首先BeanFactory接口定义出了基本的factory功能，其次再使用三个子接口继承BeanFactory，并分别从三个方面对基本的factory功能进行了扩展。上图中剩下的接口也是按照这个思路：

 - `ConfigurableBeanFactory` 定义了BeanFactory的配置接口
 - `ConfigurableListableBeanFactory` 定义了可配置的LisableBeanFactory，在普通ConfigurableBeanFactory针对ListableBeanFactory定义了一些特殊功能

接下来是这些接口的实现，最上层的实现是`AbstractBeanFactory`，它实现了`ConfigurableBeanFactory`，也就是说配置相关的代码主要在该实现中，然后`AbstractAutowireCapableBeanFactory`继承了该实现，并进一步实现了`AutowireCapableBeanFactory`，也就是说autowire相关的功能代码大部分在该抽象类中。最后是`DefaultListableBeanFactory`，继承了`AutowireCapableBeanFactory`，并进一步实现了`ConfigurableListableBeanFactory`接口。

该类图右侧的Registry系列类并不是太重要，主要是负责辅助存储一些信息，例如bean的别名、单例bean等等，这些类存在的主要意义是将这些公共任务进行抽象并从其他具体实现中分离出来，保证代码的清洁度并且进行复用。

这样大致就知道BeanFactory的主要功能代码所在的位置了，后面将会进一步深入BeanFactory的具体实现细节。