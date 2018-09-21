# springbootcloud-all

项目结构:
 ``` bash
 ├── springbootdemoall
 │   ├── client-common-dependencys
 │   ├── client-feign
 │   ├── client-gateway-zuul
 │   ├── client-order-ribbon
 │   ├── client-turbine-monitor
 │   ├── docker-compose-base.yml
 │   ├── docker-compose.yml
 │   ├── docker-start.sh
 │   ├── eureka-server
 │   ├── mvn-package-docker.sh
 │   ├── pom.xml
 │   ├── service-common-dependencys
 │   ├── service-order
 │   ├── service-user
 │   ├── spring-boot-admin-server
 │   ├── springbootdemoall.iml
 │   ├── src
 │   ├── swagger-doc
 │   └── var
 ```


搭建spring cloud的微服务，Eureka、Feign、Ribbon、Hystrix、Hystrix Dashboard、Turbine聚合监控、Zuul
除spring-boot-admin-server外，其它应用都集成了admin-client

## Hystrix
 
springboot 版本如果是2.0则需要添加 ServletRegistrationBean
因为springboot的默认路径不是 "/hystrix.stream"，
只要在自己的项目里配置上下面的servlet就可以了
如果不注入Bean，第一次访问hystrix.stream 会出现 ```Unable to connect to Command Metric Stream```
  ```java
@Bean
public ServletRegistrationBean getServlet() {
    HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
    ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
    registrationBean.setLoadOnStartup(1);
    registrationBean.addUrlMappings("/hystrix.stream");
    registrationBean.setName("HystrixMetricsStreamServlet");
    return registrationBean;
}
```
如果是Feign结合Hystrix，那么配置文件中必须加上
```xml
feign:
  hystrix:
    enabled: true
```

## Turbine
因为聚合项目中加添了有Hystrix的项目名称,所以启动的时候会去访问这些项目的/hysitrix.stream,会发生错误，找不到路径
必须在配置文件中添加配置来纠正
turbine的配置参阅过 https://segmentfault.com/a/1190000011478978
```xml
turbine:
  app-config: client-feign,client-order-ribbon,client-gateway-zuul # 指定了要监控的应用名字
  cluster-name-expression: new String("default") # 表示集群的名字为default
  combine-host-port: true # 表示同一主机上的服务通过host和port的组合来进行区分，默认情况下是使用host来区分，这样会使本地调试有问题
  instanceUrlSuffix: hystrix.stream
management:
  context-path: /
```

这里解决 /actuator/hystrix.stream 访问不了问题 http://blog.didispace.com/spring-cloud-tips-4/

Hystrix项目的访问路径 是 /hystrix /hystrix.stream
turbine项目的访问路径 是 /turbine.stream

## zuul
项目中遇到的问题

这里解决 Thread Pools 一直处于 loading中
这是由于Zuul默认会使用信号量来实现隔离，只有通过Hystrix配置把隔离机制改成为线程池的方式才能够得以展示
https://blog.csdn.net/u010826617/article/details/82260873
SEMAPHORE - 它在调用线程上执行，并发请求受信号量计数的限制(Zuul默认此策略)
THREAD - 它在一个单独的线程上执行，并发请求受到线程池中线程数的限制

```xml
thread-pool:
    use-separate-thread-pools: true
  ribbon-isolation-strategy: thread # 每个路由使用独立的线程池
```