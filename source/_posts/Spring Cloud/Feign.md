---
title: Spring Cloud Feign

date: 2019-12-03 18:00:00

categories: 
- Spring Cloud

tags:
- 教程
---

Feign是一个声明式的伪Http客户端，使用Feign，只需要创建一个接口并注解。Feign默认集成了Ribbon，并和Eureka结合，默认实现了负载均衡的效果。

- Feign采用基于接口的注解
- Feign整合了Ribbon，具有负载均衡能力
- 整合了Hystrix，具有熔断能力

<!--more-->

### 准备工作
和Ribbon工程一样， 启动Eureka Server工程，启动service-hello工程两次

### 建立Feign服务

新建一个module子工程，取名为service-feign，引入依赖：

```
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>pers.hogwarts</groupId>
        <artifactId>cloud</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>


    <artifactId>cloud-server-feign</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>cloud-server-feign</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>

        <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
          <version>2.1.1.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
            <version>RELEASE</version>
        </dependency>

    </dependencies>
```

配置文件如下：
```

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
server:
  port: 8765
spring:
  application:
    name: service-feign

```

在启动类加上@EnableFeignClients注解开启Feign功能

```
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class ServerFeignApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServerFeignApplication.class, args);
    }
}
```

定义一个接口，通过注解@FeignClient("服务名")来决定调用哪个服务

```
@FeignClient(value = "service-hello")
public interface HelloServiceClient {

    @GetMapping("/hello")
    String hello(@RequestParam("name") String name);
}


@RestController
public class HelloController {

    @Autowired
    private HelloServiceClient helloService;


    @RequestMapping(value = "/hello")
    public String hi(@RequestParam String name){
        return helloService.hello(name);
    }
}
```


启动工程， 打开浏览器输入 localhost:8765/hello?name=feign 多次访问可以看到浏览器交替显示

> hello feign, i am from port:8762

> hello feign, i am from port:8763

此时的架构图与Ribbon一样。在服务间调用方面，Feign可以看做基于Ribbon的封装，调用更加方便灵活，但本质上是一样的。


### 在Feign中开启断路器

在微服务间，如果由于意外导致某个服务不可用，由于服务间的依赖，故障传播，可能会影响到整个微服务体系，为了解决这个问题，就要使用断路器熔断降级。

Feign中已经集成了Hystrix，默认是关闭的所以要在配置文件中开启

```
feign:
  hystrix:
    enabled: true
```

开启Feign后只要在Feign Client中指定fallback类即可


```
@FeignClient(value = "service-hello", fallback = HelloServiceFallback.class)
public interface HelloServiceClient {

    @GetMapping("/hello")
    String hello(@RequestParam("name") String name);
}

@Component
public class HelloServiceFallback implements HelloServiceClient {

    @Override
    public String hello(String name) {
        return "sorry " + name;
    }
}
```

启动Feign工程，关闭service-hello服务，打开浏览器输入 localhost:8765/hello?name=feign

> sorry，feign

注意：如果在fallback中抛出自定义异常无法捕获，因为会先引发出Hystrix中的异常