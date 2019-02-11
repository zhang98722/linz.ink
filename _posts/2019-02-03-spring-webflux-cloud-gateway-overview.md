---
layout: post
title:  初识Spring WebFlux和Spring Cloud Gateway
date:   2019-02-03
categories: 
  - 网关
  - Gateway
  - Spring Cloud
  - WebFlux
comments: false
---

## Overview

#### Spring WebFlux

随着NIO（以Netty为代表）编程模型的兴起，同时函数式（lambda express）、响应式编程（rxJava，projectReactor）生态的慢慢建立，越来越多的程序员开始慢慢熟悉这种不同于以往命令式的编程模型，Spring在5.0时代推出了Spring WebFlux来替代以往基于Servlet模型的Web框架，同时也发布了Spring Cloud Gateway以替代之前的Spring Cloud Zuul。

<!-- more -->

相当于Spring MVC，Spring WebFlux具有如下优势：

- 非阻塞模型，能够用更少的资源（主要是线程），支持更高的并发
- 支持函数式编程，类似于jdk1.5的注解，极大的解放了编程的生产力
- 通过良好的设计，编程模型更简洁

当然这些优势也不是没有成本的：

- 需要继承Servlet的生态（例如RequestMapping这种注解），有额外的工作量，当然这也有助于用户平稳过度，方便接受新的编程模型
- 非阻塞、响应式的编程模型在开发、调试、诊断、调优等过程中难度大大增加，增加了对于开发人员的要求
- 阻塞式API的资源消耗反而增加

但是Spring WebFlux并不能让你的程序运行得更快，它更多的是让你的程序在并发支持上面有更强大的支持，如果你没有这方面的需求，易于编写和调试的Spring MVC更适合你，而如果你需要使用阻塞式API，Spring WebFlux甚至会让你的承载能力更低。

#### Spring Cloud Gateway

Spring Cloud Gateway已经是Spring Cloud的基于第二代网关实现了，从代码来看，Gateway的编程模型和Zuul非常相似，均是以RequestMapping作为切入点，将外部请求的流量导入到内部，再通过Filter Chain对请求一步一步进行处理，最终将请求转发到原始接口；而区别则是Zuul是通过RxJava将请求包装成HystrixCommand来执行，而Gateway转而使用ProjectReactor，进行更直接的响应式编程。

当然，Spring Cloud Gateway花大气力从Zuul重构，并不是没有意义，在实际测试中我们会发现基于Servlet的Zuul对于资源的耗用是非常高，对于并发的支持并不是很好，平均延迟在10ms的接口，经过Zuul代理之后，在2000~3000QPS之前线程数量和能够承载的QPS基本是线性关系，而在3000QPS之后，增加线程数量基本很难再提高QPS承载。而Gateway对于并发的承载并不是线性关系，性能相比zuul至少有1.5倍的提升，未来随着HTTP/2，应该还有更大的潜力。

当然，Spring Cloud Gateway作为一个网关的core是ok的，但是确实配套的生态环境，他的角色更多是类似于Nginx-OpenResty-Kong体系中的OpenResty，如果真正想依托于Gateway建立一个真正的网关，还需要自己构建一个配置、管理、监控的中心。

## Spring WebFlux

Spring WebFlux作为Spring框架中的下一代Web框架，但是其中有很多和Spring MVC相同的地方，参考下图：

![spring-mvc-and-webflux-venn](/images/20190203/spring-mvc-and-webflux-venn.png)

我们可以看到，Spring WebFlux继承了Spring MVC的注解和容器，同时新一代的响应式HttpClient也可以用于Spring MVC。同时，对于阻塞式API，Spring WebFlux是没有原生支持的，虽然可以通过技术手段来做同异步转换，但是反而消耗更多的资源（Zuul就是这样做的），失去了响应式编程的意义。

为了方便理解，我们通过和Spring MVC对比来理解Spring WebFlux：

- 容器
    - 相同
        - 负责管理TCP连接，并且将TCP Socket中的字节流反序列化成Request，同时在后端处理完毕后将Response序列化为字节流写入TCP Socket
    - 差异
        - Spring MVC的容器是通过Servlet接口将封装好的HttpServletRequest和HttpServletResponse交给Web框架（_org.springframework.web.servlet.DispatcherServlet_），而Spring WebFlux则会将请求封装为ServerHttpRequest和ServerHttResponse，再交给Web框架（_org.springframework.http.server.reactive.HttpHandler_）处理，最后再进行进一步的抽象，封装ServerWebExchange（_org.springframework.web.server.ServerWebExchange_）交给WebHandler型执行（_org.springframework.web.reactive.DispatcherHandler_）
- 处理模型
    - 相同
        - 为了方便扩展，两者均通过FilterChain来进行后端流程的处理
        - 均支持通过注解来标记Http请求的处理逻辑（RequestMapping）
        - 处理流程基本一样，请求 -> requestMapping- > 处理
    - 不同
        - Spring WebFlux为了同时支持遗留的Spring MVC模型和新的函数式处理模型，增加了新的HandlerAdapter，通过不同的adapter，来支持不同的编程模型，所以既可以和MVC一样通过注解配置将请求委托给某个method处理，也可以委托给webHandler
- 编程范式
    - 不同
        - 一个是传统的命令式编程，一个是依托于ProjectReactor的响应式编程

两者的异同通过他们的最核心的类DispatchServlet（Spring MVC）和DispatcherHandler（Spring WebFlux）就能得到足够体现。

#### Spring MVC源码解析

```
//org.springframework.web.servlet.DispatcherServlet#doDispatch(line:932)（Spring MVC）
ModelAndView mv = null;
Exception dispatchException = null;

try {
	processedRequest = checkMultipart(request);
	multipartRequestParsed = (processedRequest != request);

	// Determine handler for the current request.
	mappedHandler = getHandler(processedRequest);
	if (mappedHandler == null || mappedHandler.getHandler() == null) {
		noHandlerFound(processedRequest, response);
		return;
	}

	// Determine handler adapter for the current request.
	HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

	// Process last-modified header, if supported by the handler.
	String method = request.getMethod();
	boolean isGet = "GET".equals(method);
	if (isGet || "HEAD".equals(method)) {
		long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
		if (logger.isDebugEnabled()) {
logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
		}
		if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
return;
		}
	}

	if (!mappedHandler.applyPreHandle(processedRequest, response)) {
		return;
	}

	// Actually invoke the handler.
	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

	if (asyncManager.isConcurrentHandlingStarted()) {
		return;
	}

	applyDefaultViewName(processedRequest, mv);
	mappedHandler.applyPostHandle(processedRequest, response, mv);
}
```
Spring MVC首先通过getHandler获取保存在handlerMapping的调用链（filter之类），然后使用handlerMapping中的HandlerAdapter来进行真正执行handler，在之前和之后执行调用链的pre和post，得到ModelAndView，最终再将ModelAndView通过渲染器渲染，得到最终的结果。

#### Spring WebFlux源码解析

```
//org.springframework.web.reactive.DispatcherHandler#handle(line:147) Spring WebFlux
return Flux.fromIterable(this.handlerMappings)
			.concatMap(mapping -> mapping.getHandler(exchange))
			.next()
			.switchIfEmpty(createNotFoundError())
			.flatMap(handler -> invokeHandler(exchange, handler))
			.flatMap(result -> handleResult(exchange, result));
```
基本流程和Spring MVC类似，通过遍历handlerMapping获取可用的handlerMapping，在进行调用，最后处理调用结果，这里已经可以明显感觉到函数式编程和传统响应式编程的区别：
- 核心流程非常简洁，当然成本也是在其他地方必须有充足的准备才可以
- 运行时基本分成两个阶段：a）调用流程的组装，即通过函数式编程构建整个执行链，这个阶段只是调用流程的组装，并没有正在的执行；b）通过触发执行整个执行链。这样的编程模式导致基本没有办法进行传统的调试：在构建执行链的时候没有数据，在最终执行的时候又没有执行链的结构。


总体来说，Spring MVC和Spring WebFlux是同一设计思想的两种不同的实现，适配的是不同的适用场景：

- Spring MVC能够适配我们现在99%以上的场景，编程模型简单但是足够解决问题
- Spring WebFlux在更适合高并发中或者高延迟的环境下使用，但是由于支持非阻塞响应式编程的组件太少（目前我仅发现WebClient），在相关生态环境没有完善之前，很难有大的作为

全流程的非阻塞响应式编程目前来看，一方面是很多场景并不需要这么复杂的编程模型，已有的已经足够使用，另一方面生态环境和从业人员都还不够，未来道阻且长。

## Spring Cloud Gateway

Spring Cloud Gateway作为Spring Cloud的第二代网关，除了在响应式编程方面有很多变化，其他的基本类似：
- 自己不提供web容器，依托于Spring的生态（Zuul依托于Spring MVC，gateway依托于Spring WebFlux）
- 编程模型简洁明了，均是把自己封装成普通的web服务，通过HandlerMapping将请求映射到自己的web服务，再通过filter chain对请求进行封装，再通过httpClient（zuul主要是ApacheHttpClient和OKHttpClient，而gateway是webClient）将请求转发
- 集成Spring Cloud

这个从Spring Cloud Gateway Core的目录结构就能一窥一二：
```
-- org.springframework.cloud.gateway
---- + actuate              //支持actuate，并提供一定的控制命令
---- + config               //支持SpringBoot
---- + discovery            //支持Spring Cloud的服务发现
---- + event                //响应全局事件,例如刷新配置等等
---- + filter               //调用链
---- + handler              //与Spring WebFlux进行继承的部分
---- + route                //负责路由规则封装规则,用于匹配外部请求来将流量引入Gateway
---- + support              //一些辅助工具类
```

值得关注的是有Gateway特色的WebClient部分,这里需要一分为二:
- 直接配置的请求转发规则直接使用webClient进行请求的转发,详细参考 _org.springframework.cloud.gateway.filter.WebClientHttpRoutingFilter_
- 对于通过Spring Cloud进行服务发现的转发,则是通过 _org.springframework.cloud.gateway.filter.LoadBalancerClientFilter_ 和 _org.springframework.cloud.gateway.filter.NettyRoutingFilter_ 进行webClient的底层直接通过netty转发

而在filterChain方面，Zuul将filter分为pre，route，post三种类型，分别用于请求转发之前，转发请求，请求转之后三个阶段，再通过排序组成3个调用链，用于在3个阶段分别执行，而gateway只有一条调用链，执行顺序通过全局的排序顺序执行，这里也可以看出作者一直在尝试将代码尽可能的简化的方向。

总体来说,Spring Cloud Gateway或者Zuul本身来说作为网关的承载是完全没有问题,缺少的是配套的生态环境，类似于OpenResty的Kong。