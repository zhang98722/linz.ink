---
layout: post
title:  SpringCloud微服务最佳实践
date:   2018-11-12
categories: 
  - Spring Cloud
  - 微服务
---
### 技术选型

参考[SpringCloud调研报告](http://note.youdao.com/noteshare?id=0dcbb7a6020182021541550a2bda98bc&sub=7D5FD3A5E4684D48B02502B93706E125)，我们的技术选型为：

 - 注册中心：Consul集群 （为后续的部署和服务分离做准备）
 - RPC组件：Ribbon + Hystrix + OpenFeign + Apache Httpclient （标准使用）
 - 监控管理：Sleuth + [SkyWalker](https://github.com/apache/incubator-skywalking) + SpringBootAdmin （JVM监控、方法监控、链路跟踪、JMX远程管理）

<!-- more -->

### 具体实施

#### Consul集群搭建

Consul集群由一个boostrap实例（一个特殊的server实例） + 多个server实例 + 没有或者多个client实例（主要用来做监控管理）组成。

Consul默认会使用以下端口，所以如果有防火墙的话需要按需打开：
 - 8300：agent服务relplaction、rpc（client-server）
 - 8301：lan gossip
 - 8302：wan gossip
 - 8500：http api端口
 - 8600：DNS服务端口

#### bootstrap实例

```
consul agent  -bootstrap-expect 1  -server -ui -log-file /export/log/consul/consul.log -data-dir /export/data/consul -node=node1 -bind=10.28.6.41 -client 10.28.6.41 -config-dir /export/data/consul -enable-script-checks=true  -datacenter=defaul
```
参数解析：
 - -bootstrap-expect：最小适用集群，建议至少为集群总数量/2+1
 - -server：作为server节点
 - -ui：提供ui服务
 - -log-file：指定日志目录
 - -data-dir：数据临时目录
 - -node：节点名称
 - -bind：consul服务绑定ip
 - -client：ui服务绑定ip，默认为127.0.0.1，只能本机访问
 - -config-dir：配置临时目录
 - -enable-script-checks：健康检查
 - -datacenter：所属数据中心

#### server实例

```
consul agent -server -log-file /export/log/consul/consul.log -data-dir /export/data/consul -node=node2 -bind=10.28.6.79 -config-dir /export/data/consul -enable-script-checks=true  -datacenter=default  -retry-join 10.28.6.41
```

参数解析：
 - -retry-join：加入到的集群ip，失败自动重试
 
### SpringCloud实现

SpringCloud作为微服务的一种，其实在实施规范上和Dubbo有很多共通之处，首先可以参考 [Dubbo最佳实践](http://dubbo.apache.org/zh-cn/docs/user/best-practice.html) 。

#### 分包

首先是最重要的分包，为了统一规范接口设计，同时也降低接入成本，虽然SpringCloud官方不推荐使用分包的策略共享api的domain，但是在实际实施中，基本都会共享api和domain（如果能够规范异常更好，能够形成统一的异常处理规范）。

建议的共享分包目录结构为：

```
 + -- api
    + -- package-service-a
            api-a-1
            api-a-2
    + -- namespace-b
            api-b-1
            api-b-2
 + -- domain
    + -- package-domain-a
            domain-a-1
            domian-a-2
    + -- package-domain-b
            domain-b-1
            domain-b-1
 +  -- exception
 +  -- util
 ```
通过共享统一的共享包来进行接口的对接，保证一致，同时通过共享包的版本控制，来确定服务的版本。

在接口规范上面，由于我们均是通过RestFul接口提供服务，所以可以在定义接口的时候，就确定各个具体接口的path的method，那么在服务提供和服务消费的时候就能够进行统一，参考：

```
@RequestMapping(value = "/serviceA")
public interface ServiceA {

    @RequestMapping(value = "/invokeB", method = RequestMethod.GET)
    TrackInfo                   invokeB();
    
    ...

}
```

#### 服务发布

##### SpringMVC实现

参考之前的分包策略，服务方在实现业务的时候，只需要实现对应的接口，即可满足规范，但是需要注意的是 **SpringMVC在做接口映射的时候，使用的是标准的Spring反射流程，所以没有实现对方法参数注解的继承，在实现方法的时候，还需要对方法参数的注解进行补填**，参考：
```
@RestController
public class ServiceBController implements ServiceB {

    @Override
    public TrackInfo stop(@RequestBody TrackInfo trackInfo) {
            ...
        return trackInfo;
    }
}
```

##### 服务发现

通过引入*spring-cloud-starter-consul-all*之后，服务发现基本能够达到开箱即用，不过SpringCloud的传统策略就是使用主机名作为应用实例，对于容器可能会遇到主机名重复的情况，参考配置如下：

```
spring:
  application:
    name: ${serviceId}
  cloud:
    consul:
      host: 10.28.6.79
      port: 8500
      discovery:
        ##默认关闭，为主机名
        prefer-ip-address: true
        ##默认为服务名+端口，没有主机名
        instance-id: ${serviceId}:${spring.cloud.client.ip-address}:${server.port}
```

#### 服务消费

#### 服务发现

作为消费方，服务发现基本和服务方一致，只需要依赖*spring-cloud-starter-consul-all*即可。

#### RPC调用

由于之前进行了分包，所以在消费时候代码实现是非常方便的，直接使用继承+FeignClient即可：

```
@FeignClient(value = "${serviceId.serviceB}")
public interface ServiceBClient extends ServiceB{
}
```

但是OpenFeign/Ribbon/Hystrix都有大量的配置，都可能需要根据实际情况进行调整，下面是一些默认配置：

OpenFeign
```
feign:
  httpclient:
    #单点最大并发连接数，集群越大，连接数需要的越少，反之
    max-connections-per-route: 10
    #客户端最大并发连接数，和集群规模有关
    max-connections: 500
  hystrix:
    #开启Hystrix
    enabled: true
  #开启压缩
  compression:
    request:
      enabled: true
    response:
      enabled: true
```

Ribbon
```
ribbon:
  eager-load:
    #饥饿模式，启动即开始加载，需要配合clients
    enabled: true
  http:
    client:
      #启用Apache的httpclient代替默认的URLConnection
      enabled: true
```

Hystrix
```
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            #线程最大阻塞时间
            timeoutInMilliseconds: 10000
  threadpool:
    default:
      coreSize: 10
      allowMaximumSizeToDivergeFromCoreSize: true
      maximumSize: 50
      maxQueueSize: 10
      keepAliveTimeMinutes: 5
```

这里需要注意的是，当消费方本身又提供SpringMVC服务时，SpringMVC会在FeignClient加载的时候将之加入HandlerMapping中，因为我们在做分包的时候，给FeignClient都加入了RequestMapping，为了避免这个，需要针对SpringMVC做一个额外的配置，来屏蔽FeignClient接口中的RequestMapping，示例如下：

```
    /**
     * 在Spring bean被初始化的时候，默认会检查是否有RequestMapping，如果有则会加入SpringMVC的适配中，
     * 而feignClient通过接口基础的时候明显是不需要加入对外是SpringMVC服务，所以需要过滤一下
     * @return
     */
    @Bean
    public WebMvcRegistrations feignWebRegistrations() {
        return new WebMvcRegistrations() {
            @Override
            public RequestMappingHandlerMapping getRequestMappingHandlerMapping() {
                return new RequestMappingHandlerMapping(){
                    @Override
                    protected boolean isHandler(Class<?> beanType) {
                        return super.isHandler(beanType) &&
                                !AnnotatedElementUtils.hasAnnotation(beanType, FeignClient.class);
                    }
                };
            }
        };
    }
```

### 其他

#### actuator

actuator是一个不错的组件，对于查看和管理线上实时运行状态非常有帮助，但是如果对外暴露则有较大的安全隐患，所以参考调研报告，要么整体SpringSecurity进行鉴权，否则最好通过端口分离隔离：
```
management:
  server:
    port: 8101
```
或者也可以将之暴露简单的health接口用于进行监控检查，其他的服务全部关闭。