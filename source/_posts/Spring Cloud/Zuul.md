---
title: Spring Cloud Zuul

date: 2020-01-12

categories: 
- Spring Cloud

tags:
- 教程
---

在Spring Cloud微服务系统中，一种常见的负载均衡方式是，客户端的请求首先经过负载均衡（zuul、Nginx），再到达服务网关（zuul集群），然后再到具体的服务。

<!--more-->

### 简介

Zuul的主要功能是路由转发和过滤器。路由功能是微服务的一部分，比如／api/user转发到到user服务，/api/shop转发到到shop服务。zuul默认和Ribbon结合实现了负载均衡的功能。

### 创建工程

引入zuul依赖

```
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>
    </dependencies>
```

在启动类加上@EnableZuulProxy注解表示开启Zuul网关

在配置文件加上以下配置
```
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
server:
  port: 8769
spring:
  application:
    name: service-zuul
zuul:
  routes:
    api-a:
      path: /ribbon/**
      serviceId: service-ribbon
    api-b:
      path: /feign/**
      serviceId: service-feign
```

和其它module一样都注册到Eureka， 以/ribbon/开头的请求都转发给 service-ribbon服务，以/feign/开头的请求都转发给service-feign。

将以上服务全部开启，浏览器访问打开浏览器访问：http://localhost:8769/ribbon/hello?name=zuul
> hello zuul,i am from port:8762

打开http://localhost:8769/feign/hello?name=zuul 也能看到zuul转发成功

### 过滤

zuul除了网关功能外还可以进行过滤，做一些校验。
```
@Component
public class MyZuulFilter extends ZuulFilter {

    /**
     * 过滤器类型
     * pre： 路由之前
     * routing： 路由之时
     * post： 路由之后
     * error： 发送错误调用
     */
    @Override
    public String filterType() {
        return "pre";
    }

    /**
     * 过滤的顺序
     */
    @Override
    public int filterOrder() {
        return 0;
    }

    /**
     * 过滤条件
     */
    @Override
    public boolean shouldFilter() {
        return true;
    }

    /**
     * 过滤器具体逻辑
     * @return
     * @throws ZuulException
     */
    @Override
    public Object run() throws ZuulException {

        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        Object accessToken = request.getParameter("token");

        if(accessToken == null) {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            try {
                ctx.getResponse().getWriter().write("token is empty");
            }catch (Exception e){
                e.printStackTrace();
            }

            return null;
        }

        return null;
    }
}

```

新建一个类继承ZuulFilter，run()方法里是执行过滤的逻辑，如果Filter类中的doFilter()方法，shouldFilter()方法是判断是否过滤，可以自定义过滤条件。

这个时候在浏览器输入 http://localhost:8769/ribbon/hello?name=zuul 可以看到
> token is empty
