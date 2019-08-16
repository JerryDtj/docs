# Spring cloud——openfeign教程

## 是什么

一种服务的调用工具.它可以做到以接口的形式调用远方服务.可以把它看成是RestTemplate的封装版.在springCloud中也是这么做的.最后还是通过RestTemplate发起的远程调用

## 怎么用

代码下载

- pom.xml

  ```xml
  <!--       心跳服务-->
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-actuator</artifactId>
          </dependency>
  <!--        服务注册，用来注册到consul上-->
          <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-consul-discovery</artifactId>
          </dependency>
  <!--        springfeign-->
          <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-openfeign</artifactId>
          </dependency>
  ```

- application.yml

  ```yaml
  server:
    port: 10020
  spring:
    application:
      name: consul-feign
    cloud:
      consul:
  #      consul服务端地址
        host: localhost
        port: 8500
  #      下面的配置主要是用于consul的心跳检测。如果不用发布服务，那么可以不配置
        discovery:
          health-check-url: http://localhost:10020/actuator/health
          service-name: consul-feign
      discovery:
        client:
  #        是否开启心跳检测
          health-indicator:
            enabled: true
  ```

- ConsulConsumerFeignApplication.java

  ```java
  package com.learn.consulconsumerfeign;
  
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
  import org.springframework.cloud.openfeign.EnableFeignClients;
  
  /**
   * despaction
   *
   * @Author: jerry
   * @date: 2019/8/8 18:01
   * @description:
   **/
  @SpringBootApplication
  @EnableDiscoveryClient
  @EnableFeignClients
  public class ConsulConsumerFeignApplication {
      public static void main(String[] args) {
          SpringApplication.run(ConsulConsumerFeignApplication.class,args);
      }
  }
  ```

- dcService.java

  ```java
  package com.learn.consulconsumerfeign.service;
  
  import org.springframework.cloud.openfeign.FeignClient;
  import org.springframework.web.bind.annotation.GetMapping;
  
  /**
   * consul-client 服务端
   *
   * @author Jerry
   * @Date 2019-08-12 08:56
   */
  @FeignClient("consul-client")
  public interface DCService {
  
      @GetMapping("/dc")
      String getDC();
  }
  
  ```

- JerryService.java

  ```java
  package com.learn.consulconsumerfeign.service;
  
  import org.springframework.cloud.openfeign.FeignClient;
  import org.springframework.web.bind.annotation.GetMapping;
  
  /**
   * despaction
   *
   * @Author: jerry
   * @date: 2019/8/12 10:52
   * @description:
   **/
  @FeignClient("consul-client-jerry")
  public interface JerryService {
      @GetMapping("/dc")
      public String getDc();
  }
  ```

- DcController.java

  ```java
  package com.learn.consulconsumerfeign.controller;
  
  import com.learn.consulconsumerfeign.service.DCService;
  import com.learn.consulconsumerfeign.service.JerryService;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.web.bind.annotation.GetMapping;
  import org.springframework.web.bind.annotation.RestController;
  
  /**
   * 服务消费者
   *
   * @author Jerry
   * @Date 2019-08-12 09:04
   */
  @RestController
  public class DcConsumer {
      @Autowired
      private DCService dcService;
  
      @Autowired
      private JerryService jerryService;
  
      @GetMapping("/dc")
      public String dcConsumer(){
          String dc = dcService.getDC();
          String jerryDc = jerryService.getDc();
          return String.format("return:%s,%s",dc,jerryDc);
      }
  }
  ```

  