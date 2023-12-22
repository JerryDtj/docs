# springCloud——hystrix使用

## 是什么

是微服务框架中必备的一个组件.为了防止由于某单个服务发生问题,导致的雪崩效应,springcloud 特地引入了这个熔断机制.

主要分为Hystrix客户端和监控:dashBoard监控面板2部分组成.本文主要介绍Hystrix客户端的使用.

# Hystrix的使用方法

[代码下载](https://github.com/JerryDtj/springcloudgreenwich/tree/master/consul-hystrix-consumer)

### Pom.xml

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
```

Application.yml

```yaml
server:
  port: 10040
spring:
  application:
    name: consul-consumer-hystrix
  cloud:
    consul:
      discovery:
        service-name: hystrix-consume
```

Application.java

```java
package com.learn.hystrix;

import com.netflix.hystrix.contrib.metrics.eventstream.HystrixMetricsStreamServlet;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.cloud.client.SpringCloudApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.context.annotation.Bean;

/**
 * @author Jerry
 * @Date 2019/8/27 9:18 下午
 */
@SpringCloudApplication
@EnableFeignClients
public class ConsulConsumerHystrixApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsulConsumerHystrixApplication.class,args);
    }

    @Bean
    public ServletRegistrationBean hystrixMetricsStreamServlet() {
        ServletRegistrationBean registration = new ServletRegistrationBean(new HystrixMetricsStreamServlet());
        registration.addUrlMappings("/hystrix.stream");
        return registration;
    }
}

```

DcController

```java
@RestContorller
public class DcConsume(){
    @Autowired
    private DcServiceImpl dcServiceImpl;

    @GetMapping("/dcConsume")
    @HystrixCommand(fallbackMethod="dcFallBack")
    public String dc(){
       return dcServiceImpl.dc();
    }

    public String dcFallBack(){
        return "Hystrix fall back";
    }
}
```