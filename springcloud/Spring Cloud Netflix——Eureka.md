# Spring Cloud Netflix——Eureka

## 来源

本文所有内容均来源于[spring cloud](https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.2.2.RELEASE/reference/html/#spring-cloud-eureka-server) 官方文档

## 简介

·Netflix下主要分为服务发现（Eureka），断路器（Hystrix），智能路由（Zuul）和客户端负载平衡（Ribbon）几个模块.本文主要讲述Eureka模块.

Eureka 主要分为server端和client端

## server端使用

1. pom.xml 引入

   ```xml
   <dependencies>
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
       </dependency>
       <dependency>
           <groupId>org.projectlombok</groupId>
           <artifactId>lombok</artifactId>
       </dependency>
   </dependencies>
   ```

2. 启动类

   ```java
   /**
    * @author Jerry
    * @Date 2020/5/1 7:12 上午
    */
   
   @EnableEurekaServer
   @SpringBootApplication
   public class Strat {
       public static void main(String[] args) {
           SpringApplication.run(Strat.class,args);
       }
   }
   ```

3. application.yml(以独立模式启动)

   ```yaml
   server:
     port: 8080
   spring:
     application:
       name: eureka1
   eureka:
     instance:
       hostname: localhost
     client:
       registerWithEureka: false
       fetchRegistry: false
       serviceUrl:
         defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
   ```

## client端使用

1. pom.xml

   ```xml
   <dependencies>
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
       </dependency>
       <dependency>
           <groupId>org.projectlombok</groupId>
           <artifactId>lombok</artifactId>
       </dependency>
   </dependencies>
   ```

2. 启动类

   ```java
   package com.learn.erueka.client;
   
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   
   /**
    * @author Jerry
    * @Date 2020/5/10 5:07 下午
    */
   
   @SpringBootApplication
   public class EurekaClientStart {
   
       public static void main(String[] args) {
           SpringApplication.run(EurekaClientStart.class,args);
       }
   }
   ```

3. application.yml

   ```yaml
   server:
     port: 7070
   spring:
     application:
       name: eureka-client1
   eureka:
     client:
       service-url:
         defaultZone: http://localhost:8080/eureka/
   ```

4. controller

   ```java
   package com.learn.erueka.client.controller;
   
   import lombok.extern.slf4j.Slf4j;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RestController;
   
   /**
    * @author Jerry
    * @Date 2020/5/10 5:28 下午
    */
   @RestController
   @RequestMapping("/application")
   @Slf4j
   public class HelloWord {
       @GetMapping("/name")
       public String getApplicationName(){
           log.info("client request coming");
           return "eureka-client1";
       }
   }
   ```