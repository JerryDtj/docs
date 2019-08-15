# springCloud——注册中心

## 是什么

由于原有的eureka体系重构过于复杂，所以eureka2.x版本胎死腹中。

随之替代的注册中心为consul。consul比eureka的优点如下：

​	**Eureka优点是注册速度很快，不管同步到其他节点时是否有问题，只要服务注册到主节点既代表注册成功。牺牲了一致性，但是保证了高可用性和最终一致性。即使当前节点因为一些问题没有注册成功，那么也会通过其他节点找到当前服务，返回元数据。和eureka相比，consul注册就稍慢一些了。上面提到了，consul是强一致性的，所以consul注册服务是先注册，然后同步到各节点，raft算法使consul在有一半以上的节点注册成功时才证明服务注册成功。这样保证了数据的一致性。但也因为这样，导致注册服务相对较慢，并且当主节点挂掉之后，重新选举时整个consul不可用。可以说是牺牲了一部分可用性换来的一致性。**

consul主要分为2块：server端和client端。server端的安装教程已经在上一篇文章中讲过了。这篇文章主要着重提及consul的client的用法

## 怎么用

[代码下载](https://github.com/JerryDtj/springcloudgreenwich/tree/master/consul-client)

- ### consule 生产者：consul-client，consule-clientTwo，consule-client-jerry(后面2个都是基本一样的代码)

  1. 新建springboot项目，需要引入的pom文件如下：

     ```xml
     <!--        监控系统健康情况的工具-->
             <dependency>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-starter-actuator</artifactId>
             </dependency>
     <!--        consul注册中心-->
             <dependency>
                 <groupId>org.springframework.cloud</groupId>
                 <artifactId>spring-cloud-starter-consul-discovery</artifactId>
             </dependency>
             
             <dependency>
                 <groupId>com.alibaba</groupId>
                 <artifactId>fastjson</artifactId>
                 <version>1.2.58</version>
             </dependency>
     ```

  2. application.yml

     ```yaml
     spring:
     	cloud:
     		consul:
     			#consul server端的地址和端口号
     			host: localhost
     			port: 8500
     			discovery:
     				#本服务的心跳地址以及注册到consul上的服务名称
     				health-check-url: http://localhost:8500/actuator/health
     				service-name: consul-client
     	application:
     		name: consul-client
     server:
     	port: 10013
     ```

  3. ConsulClientApplication.java

     ```java
     package com.learn.consulclient;
     
     @springBootApplication
     @EnableDiscoveryClient
     public class ConsulClientApplication(){
         public static void main(String[] args){
             SpringApplication.run(ConsulClientApplication.class,args)
         }
     }
     ```

  4. 服务暴露controller：DcController

     ```java
     /**
      * @author Jerry
      * @Date 2019-08-07 07:46
      */
     @RestController
     public class DcController {
         @Autowired
         private DiscoveryClient discoveryClient;
     
         @GetMapping("/dc")
         public String dc(){
             return "client";
         }
     }
     ```

- ### consul消费者：

  1. 新建springboot项目：consul-consumer-feign。pom文件

     ```xml
     <!--        监控系统健康情况的工具-->
             <dependency>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-starter-actuator</artifactId>
             </dependency>
     <!--        consul注册中心-->
             <dependency>
                 <groupId>org.springframework.cloud</groupId>
                 <artifactId>spring-cloud-starter-consul-discovery</artifactId>
             </dependency>
     		<!--        openFeign消费者工具-->
             <dependency>
                 <groupId>org.springframework.cloud</groupId>
                 <artifactId>spring-cloud-starter-openfeign</artifactId>
             </dependency>
     ```

  2. application.yml 

     ```yaml
     spring:
     	cloud:
     		consul:
     			host:
     			name:
     			discovery:
     				health-check-url: http://localhost:8500/actuator/health
     				service-name: consul-consumer-feign
     	application:
     		name: consul-consumer-feign
     server:
     	port: 10020
     ```

     

  3. ConsulConsumerFeignApplication.java

     ```java
     @SpringBootApplication
     @EnableDiscoveryClient
     @EnableFeignClients
     public class ConsulConsumerFeignApplication {
         public static void main(String[] args) {
             SpringApplication.run(ConsulConsumerFeignApplication.class,args);
         }
     }
     
     ```

  4. 服务方接口：dcService

     ```java
     /**
      * @author Jerry
      * @Date 2019-08-12 08:56
      */
     @FeignClient("consul-client")
     public interface DCService {
     
         @GetMapping("/dc")
         String getDC();
     }
     ```

  5. 服务方接口：JerryService

     ```java
     @FeignClient("consul-client-jerry")
     public interface JerryService {
         @GetMapping("/dc")
         public String getDc();
     }
     ```

  6. 消费者暴露接口：DcConsumer

     ```java
     /**
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

  至此，consul的生产消费者已经全部构建完毕。