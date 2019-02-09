---
layout: post
title:  从Skywalking看如何设计一个微核+插件式扩展的搞扩展框架
date:   2018-11-26
categories: 
  - SkyWalking
  - MicroCore
---

作为程序员，我们大部分的职业生涯都在追求什么？

当然是打造一个高可用、高性能、易扩展的系统，在互联网的背景下，我们还要需要应对高并发，可伸缩等互联网的特殊场景。

为了达到这个目的，在整体设计上我们有服务化/微服务/容器化等解决方案,在应用单体上我们也常用分层设计/模块化/微核插件化等设计模式，而skywalking的基本设计模式就是基于微内核+插件：通过建立标准的插件规范，通过微核控制各个插件的生命周期，为skywalking提供了强大的扩展性。

<!-- more -->

## 经典的微核案例

微核+插件在JAVA生态中最经典的应该是Eclipse:


>一、eclipse的插件模型
>
>在eclipse中，插件是在eclipse工作平台上的一个能提供某种服务的组件。一个组件就是一个对象，该对象能够在运行前被配置到一个系统中。eclipse的运行时系统就提供了这样的功能，支持组件的激活，并使一些插件集协作来提供一个无缝的开发环境。在一个eclipse实例中，一个插件被一个或一些插件运行时类或插件类所管理。也就是说，插件类提供了配置和管理插件实例的功能。一个插件类必须扩展自org.eclipse.core.runtime.Plugin,该类是一个提供了基本插件管理功能的抽象类。
>
>每个插件有一个全局唯一的标识符id,用于标识该插件。
>
>二、插件的配置和激活
>
>插件的安装很简单，只需将插件文件夹拷贝到plugins文件夹内，这样该插件就可以在适当的时候被eclipse运行时系统激活。激活意味着加载它的运行时类并将其实例化。插件类就是在插件激活startup或反激活shutdown时完成一些特殊工作的类。
>
>eclipse包含一个插件管理核心，称为”eclipse平台“或”eclipse运行时“。每次启动时会自动加载一些核心插件。其它非核心插件只有在有需求时，才被激活。
>
>eclipse模型中，一个插件可能和其它插件有下列两种关系：
>
>1.依赖require  ：该插件的运行需要其它插件的支持。
>
>2.扩展extension：该插件扩展了原插件的功能。
>
> *摘自：https://blog.csdn.net/zhangxiang_nuaa_hust/article/details/83293792*

可以看到，参考其他的具体实践，我们可以大概的确定微核+插件的主要开发模式：

 - 微核主要提供可用资源的虚拟化或者标准化的规范，暴露统一接口，供插件使用（很类似操作系统的内核，主要是通过制定规范，依托硬件驱动，为用户系统提供可用资源的标准输入输出接口
 - 插件生命周期的维护，包括插件的加载、激活、停用，特别场景可能还需要热加载的支持

## skywalking的实现

skywalking作为一个APM实现，主要是由两部分实现，agent（采集tracing和metric数据）和后端的collector（通过RPC接受agent数据然后分析汇总放入存储容器中），具体可以参考如下的架构图：

![架构图](/images/20181126/架构图.png)

而在skywalking的agent和collector的设计中，均采用了微核+插件的设计模式，保证了skywalking的扩展性。

#### skywalking-agent

skywalking的agent核心代码非常少，主要负责三个方面的业务：
 - 建立二进制增强的增强器规范
 - 建立agent本身业务以及不依赖于二进制增强的监控业务的本地服务规范
 - 对接java agent启动器，加载满足规范的增强器并通过bytebuddy进行增强，加载满足规范的本地服务并控制他们的生命周期
 
由于bytebuddy已经提供了java agent的支持，所以agent的开发主要集中在增强器规范的定制和具体实现的加载。增强器的规范可以参考下面的规范：

![插件规范](/images/20181126/插件规范.png)

最终主要提供了对增强器的适配场景（enhancePlugin）、拦截器的具体实现(interceptPint)的统一规范和标准实现。其中enhancePlugin的模式基本和增强器的设计基本一致的：

![增强器规范](/images/20181126/增强器规范.png)

而具体实现则由几个spi接口实现：
 - InstanceConstructorInterceptor：拦截增强构造方法
 - InstanceMethodsAroundInterceptor：拦截增强普通方法
 - StaticMethodsAroundInterceptor：拦截增强静态方法

 具体格式参考：
 
![切面](/images/20181126/切面.png)
 
值得关注的是在增强器的适配场景中，设计者增加了witness的配置（参考：org.apache.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine#witnessClasses），即通过检查某个class是否被加载过来做版本的适配。

#### skywalking-oap(collector)

skywalking的collector和agent的实现模式类似，不过在collector中插件被叫做module，由ModuleManager加载，ModuleProvider控制生命周期，最终不同的ModuleProvider集成在一起成为collector，最终由BootstrapFlow启动。而collector和agent不同的是collector多了一个ModuleConfig，用于服务端进行不同环境的配置需求。

而且由于collector要适配更多的技术选型，例如存储现在就可以支持用于demo的H2内存数据库，也可以使用ElasticSearch作为测试和生产环境。所以collector不仅通过module来做插件化，同时还在core约定了各个module的标准接口，最后再通过各个module真正的实现类支持这些接口，这样既能通过插件化继续扩展增加新的业务功能，同时也有标准接口，方便不同的技术选型。

## 微核+插件式扩展的适用场景

微核的这种设计模式既然这么好，那么是不是我们都应该用呢？答案明显不是。强大的扩展性，必然代码一些混乱，例如：
 - 模块之间的复杂关系
 - 模块之间的重叠和冗余实现
 - 缺少主流程引导，业务梳理困难

所以也有另外一种模式，可以参考dubbo，通过清晰的流程和边界来实现核心业务，同时预留一定的扩展SPI，通过加载不同的SPI实现来实现可扩展，参考：

![dubbo](/images/20181126/dubbo.png)