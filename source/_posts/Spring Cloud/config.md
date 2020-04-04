---
title: Spring Cloud Config

date: 2019-12-02

categories: 
- Spring Cloud

tags:
- 教程
---

### 简介

为了方便各服务的配置文件统一管理，需要分布式配置中心组件。在spring cloud中有spring cloud config组件用于统一配置，它支持配置文件加载在内存中，也支持远程git仓库配置。在组件中，分为配置的服务者和消费者。
<!--more-->
### config server

构建config server，引入依赖


```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>pers.hogwarts</groupId>
        <artifactId>cloud</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>


    <artifactId>cloud-config-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>cloud-config-server</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
    </dependencies>

</project>

```


在启动类程序加上@EnableConfigServer注解开启配置服务器功能，由于是注册到Eureka，所有还要加上@EnableEurekaClient注解。

在配置文件加上以下配置：


```
server:
  port: 8768
spring:
  application:
    name: config-server
  cloud:
    config:
      # 远程仓库分支
      label: master
      server:
        git:
          searchPaths:  # 配置仓库路径   respo
          uri: https://github.com/LinXiuke/cloud/
          # 公开仓库不需要账号密码
          username:
          password:

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

我们需要在仓库建立一个config-client-dev的文件作为 config client的配置文件

```
app_name: config-client
```

http请求地址和资源路径的映射：

- /{application}/{profile}/{label}
- /{label}/{application}-{profile}. (yml/properties)

1. 分支名可以省略，默认是master分支
2. 无论你的配置文件是properties，还是yml，只要是应用名+环境名能匹配到这个配置文件，那么就能取到
3. 如果文件名含有多个“-”，则以最后一个“-”分割{application}和{profile}，若文件名为：my-app-demo-dev.properties，则映射的url为"/my-app-demo/dev"

浏览器输入  http://localhost:8768/config-client/dev   显示文件详情
> {"name":"config-client","profiles":["dev"],"label":"master","version":"1554c6e855dfd876f8bc7757ef473782ffd0cd2b","state":null,"propertySources":[{"name":"https://github.com/LinXiuke/cloud//config-client-dev.yml","source":{"app_name":"config-client"}}]}

http://localhost:8768/config-client-dev.yml  显示配置内容
> app_name: config-client

输入 http://localhost:8768/app_name/dev   可以直接获取单个配置信息：
> {"name":"app_name","profiles":["dev"],"label":null,"version":"1554c6e855dfd876f8bc7757ef473782ffd0cd2b","state":null,"propertySources":[]}

### config client

构建config client，引入依赖


```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">


    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>pers.hogwarts</groupId>
        <artifactId>cloud</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>


    <artifactId>cloud-config-client</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>cloud-config-client</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
    </dependencies>


</project>

```



配置文件如下


```
#spring.application.name=config-client
## 远程仓库分支
#spring.cloud.config.label=master
## 选择config-client-dev文件
#spring.cloud.config.profile=dev
## 指明配置服务中心的网址
#spring.cloud.config.uri= http://localhost:8768/
#server.port=8791



# 注册中心配置
spring.application.name=config-client
spring.cloud.config.label=master
spring.cloud.config.profile=dev

eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.serviceId=config-server
server.port=8791
```

加上@EnableEurekaClient注解注册到Eureka。

### 读取配置

进行读取文件的验证测试


```
@EnableEurekaClient
@SpringBootApplication
@RestController
public class CloudConfigClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(CloudConfigClientApplication.class, args);
    }

    @Value("${app_name}")
    private String appName;

    @GetMapping("/getAppName")
    public String getAppName() {
        return appName;
    }

}
```





浏览器输入 http://localhost:8791/getAppName，显示

> config-client