# springcloud——Hystrix dashboard 

## 是什么

Hystrix dashboard 是hystrix的一个监控面板，通过这个监控面板可以很直观的了解到哪个服务组件承受了多大压力，以及组件的运行情况。

这里只做一个简单的单服务监控。关于服务信息的聚合，可以参考博文：[Hystrix监控数据聚合【Dalston版】](http://blog.didispace.com/spring-cloud-starter-dalston-5-2/)

## 怎么用

[代码下载](https://github.com/JerryDtj/springcloudgreenwich/tree/master/consul-hystrix-dashboard)

### pom.xml

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-turbine</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

### application.yml

```yaml
server:
  port: 10030
spring:
  application:
    name: consul-config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/JerryDtj/springcloudgreenwich.git
          search-paths: config-repo
```

### ConsulHystrixDashBoardApplication.java

```java
@EnableDiscoveryClient
@EnableHystrixDashboard
@SpringBootApplication
public class ConsulHystrixDashBoardApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsulHystrixDashBoardApplication.class,args);
    }
}
```

