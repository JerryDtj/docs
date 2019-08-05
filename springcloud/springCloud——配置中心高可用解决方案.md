# springCloud——配置中心高可用解决方案

本文在springCloud——配置中心基础上做进一步的延伸.为配置中心提供高可用,动态加载方案.

需要修改的内容如下:

1. 服务端config-server-eureka

   在原有的基础上添加eureka配置中心,把服务端发布到eureka注册中心中

   具体实现可以参考eureka-client的实现

2. 客户端config-client-eureka

   1. 引入eureka,把客户端加入到注册中心

   2. Pom.xml引入动态监控模块spring-boot-starter-actuator

      ```xml
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-actuator</artifactId>
      </dependency>
      ```
   
   3. 修改bootstrap.yml文件
   
      通过`eureka.client.serviceUrl.defaultZone`参数指定服务注册中心，用于服务的注册与发现，再将`spring.cloud.config.discovery.enabled`参数设置为true，开启通过服务来访问Config Server的功能，最后利用`spring.cloud.config.discovery.serviceId`参数来指定Config Server注册的服务名。这里的`spring.application.name`和`spring.cloud.config.profile`如之前通过URI的方式访问时候一样，用来定位Git中的资源。
   
      ```yaml
      server:
        port: 10083
      
      spring:
        application:
          name: config-client
        cloud:
          config:
            discovery:
              enabled: true
              service-id: config-server-eureka
            profile: dev
      
      eureka:
        client:
          service-url:
            defaultZone: http://localhost:1001/eureka/
      
      #如果不加这个，访问refresh 会报401 Full authentication is required to access this resource.错误
      management:
        security:
          enabled: false
      ```
   
   4. 在YmlController中加入@RefreshScope注解
   
      ```java
      package com.learn.configclienteureka.contorller;
      
      import org.springframework.beans.factory.annotation.Value;
      import org.springframework.cloud.context.config.annotation.RefreshScope;
      import org.springframework.web.bind.annotation.GetMapping;
      import org.springframework.web.bind.annotation.RequestMapping;
      import org.springframework.web.bind.annotation.RestController;
      
      /**
       * 这里需要添加RefreshScope注解为了修复1.5.4.RELEASE版本的springboot不会自动刷新的问题
       * 但是在1.3.7版本是可以的。这个需要研究下为什么
       *
       * @Author: jerry
       * @date: 2019/8/5 16:39
       * @description:
       **/
      @RestController
      @RequestMapping("/yml")
      @RefreshScope
      public class YmlController {
      
          @Value("${info.add}")
          private String add;
      
          @GetMapping("/add")
          public String info() {
              System.out.println(add);
              return add;
          }
      }
      ```
   
3. 调用`http://localhost:10083/yml/add`可以得到返回结果dev

4. 动态刷新调用过程如下:

   1. 重新启动config-clinet，访问一次`http://localhost:10083/yml/add`，可以看到当前的配置值
   2. 修改Git仓库`config-repo/config-client-dev.yml`文件中`add`的值
   3. 再次访问一次`http://localhost:10083/yml/add`，可以看到配置值没有改变
   4. 通过POST请求发送到`http://localhost:10083/refresh`，我们可以看到返回内容如下，代表`add`参数的配置内容被更新了

   ```json
   [
       "info.add"
   ]
   ```

   - 再次访问一次`http://localhost:10083/yml/add`，可以看到配置值已经是更新后的值了

   

   

