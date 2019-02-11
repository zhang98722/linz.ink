---
layout: post
title:  剖析Spring Cloud Gateway
date:   2019-02-08
categories: 
  - 网关
  - Gateway
  - Spring Cloud
comments: false
---

> 这是一篇深度剖析Spring Cloud Gateway的文章，需要对Spring WebFlux和Spring Cloud Gateway有一定的理解，如果没有请参考 [初识Spring WebFlux和Spring Cloud Gateway](http://note.youdao.com/noteshare?id=eac481cb367d2f92fd7286d27c2ec72a&sub=4311F9559E014FF5AB0BDB2EA4537E18)

Spring Cloud Gateway（以下简称Gateway）是依托于Spring WebFlux的Spring Cloud网关承载的组件，主要负责发布或者聚合接口（对内或者对外的网关），本身业务模型比较简单，以下将从几个方面来对其进行剖析。

## 核心模型

作为Spring WebFlux的扩展，Gateway只是在原有的基础上通过RoutePredicateHandlerMapping拓展出来一个新的分支，具体结构可以参考下图：

![image](/images/20190208/spring-cloud-gateway-detail.png)

其中，Predicate，GatewayFilter，Route一起构成了Gateway的基础：
- Predicate用于判断请求是否由Gateway处理和由什么Route来处理
- GatewayFilter是一个个单独的处理步骤
- Route是最小的处理单元，包含由Predicate来判断是否需要自身来处理以及一系列GatewayFilter来表明处理流程

一个Route即可以认为是一个接口，当一个外部请求到来时，首先会递交到RoutePredicateHandlerMapping，handlerMapping收到请求后，会通过RouteLocator轮询所有的Route，来判断是否需要通过Gateway来处理，一旦命中某个Route，则开启处理流程，将上下文信息交给FilteringWebHandler，而FilteringWebHandler则会构建调用链并执行，将这个请求处理完毕并返回结果。

## 初始化

通过上述模型我们发现，Gateway本身并没有复杂的业务处理流程，核心业务也很稳定，在运行时基本没有或者不需要什么变化，所以基本上一旦熟悉了Gateway的初始化流程，知道了Gateway是怎么一步步构建起来上述的核心模型，基本就了解了Gateway的大部分业务。

首先我们参考一个标准的Gateway路由配置：

```
spring:
  cloud:
    gateway:
      default-filters:
      - PrefixPath=/httpbin
      - AddResponseHeader=X-Response-Default-Foo, Default-Bar
      routes:
      - id: discovery_test
        uri: lb://SERVER
        predicates:
        - Path=/api/**
        filters:
        - RewritePath=/api/SERVER/(?<segment>.*), /$\{segment}
      - id: websocket_test
        uri: ws://localhost:9000
        order: 9000
        predicates:
        - Path=/echo
```

可以看到，配置分为两部分：
- 全局的default-filter：适配所有的处理流程
- routes：多个路由规则，包含了适配规则（predicates）和处理流程（filters）

是不是有点serverless或者决策树的感觉。

整个配置会被SpringBoot适配成为 _org.springframework.cloud.gateway.config.GatewayProperties_ ，其类图可以参考：

![image](/images/20190208/spring-cloud-gateway-class.png)

可以看到GatewayProperties里面仅仅是保存了简单的Definition信息，并没有真正实例化出实际可用的运行模型，而这个任务主要是由 _org.springframework.cloud.gateway.route.RouteDefinitionRouteLocator_ 来完成：

```
	@Override
	public Flux<Route> getRoutes() {
		return this.routeDefinitionLocator.getRouteDefinitions()
				.map(this::convertToRoute)
				//TODO: error handling
				.map(route -> {
					if (logger.isDebugEnabled()) {
						logger.debug("RouteDefinition matched: " + route.getId());
					}
					return route;
				});
	}
```

以及：

```
	private Route convertToRoute(RouteDefinition routeDefinition) {
		AsyncPredicate<ServerWebExchange> predicate = combinePredicates(routeDefinition);
		List<GatewayFilter> gatewayFilters = getFilters(routeDefinition);

		return Route.async(routeDefinition)
				.asyncPredicate(predicate)
				.replaceFilters(gatewayFilters)
				.build();
	}
```

RouteDefinitionRouteLocator通过遍历RouteDefinition，通过convertToRoute方法将每个RouteDefinition转化为单个真正可用的Route；而convertToRoute主要做了三件事：
- combinePredicates：通过predicate的名字匹配所有用到的predicate工厂（通过 _private AsyncPredicate<ServerWebExchange> lookup(RouteDefinition route, PredicateDefinition predicate)_ )并实例化predicate实例，并将它们通过运算符 _and_ 串联起来，组成一个断言的链条
- getFilters：通过filter的名字匹配所有用到的filter工厂（通过 _private List<GatewayFilter> loadGatewayFilters(String id, List<FilterDefinition> filterDefinitions)_ ）并实例化filter实例
- asyncBuildRoute：将获得的predicates和fitlers加上id，uri等信息组装成真正可用的route

这里就涉及到Gateway的核心模式，所有的predicate和filter都没有具体的实例，而是只有工厂模式的工厂，当需要的时候，再通过工厂实时构建匿名函数+config组成实际需要的predicate或者filter。而RouteDefinitionRouteLocator将所有这些工厂通过SpringBoot收集起来（参考 _org.springframework.cloud.gateway.config.GatewayAutoConfiguration_ 中RouteDefinitionRouteLocator的初始化），组成对应的这些工厂的工厂（例如Map<String, GatewayFilterFactory> gatewayFilterFactories），其中key为对应的工厂的名字，而这就衍生出来Gateway中各个组件的命名规范：

```
public class NameUtils {
	public static final String GENERATED_NAME_PREFIX = "_genkey_";

	public static String generateName(int i) {
		return GENERATED_NAME_PREFIX + i;
	}

	public static String normalizeRoutePredicateName(Class<? extends RoutePredicateFactory> clazz) {
		return removeGarbage(clazz.getSimpleName().replace(RoutePredicateFactory.class.getSimpleName(), ""));
	}

	public static String normalizeFilterFactoryName(Class<? extends GatewayFilterFactory> clazz) {
		return removeGarbage(clazz.getSimpleName().replace(GatewayFilterFactory.class.getSimpleName(), ""));
	}

	private static String removeGarbage(String s) {
		int garbageIdx = s.indexOf("$Mockito");
		if (garbageIdx > 0) {
			return s.substring(0, garbageIdx);
		}

		return s;
	}
}
```

即各个predicate或者filter的key均是他们自身class的simpleName去掉对应的predicate接口或者filter接口的接口class的simpleName，如果名字一样则可能导致冲突。

最终，通过这一系列的操作，Gateway将配置中的信息全部转化为Route，构建出了核心模型，而如果是通过函数式编程来构建Route则是其中的一个子集：直接通过Route的builder构建出来route，而不需要通过各种definition绕一圈。

## 扩展业务

Gateway在运行时也不完全是一成不变的，否则没法支撑起一个可扩展的平台，Gateway也通过ApplicationEvent的机制（可以简单的认为是一个EventBus或者订阅发布模型）来进行配置的动态管理，你可以构建对应的event，然后通过发布或者消费对应的event，来进行配置的动态管理，详情可以参考core中的包： _org.springframework.cloud.gateway.event_ ，其中就实现了一些event的范例。