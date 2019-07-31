# Spring cloud 学习之feign

## 一、feign是什么

![屏幕快照 2019-06-30 下午7.32.12](/Users/dengtianjiao/Documents/java/springcloud/feign/屏幕快照 2019-06-30 下午7.32.12.png)

[开源地址](https://github.com/OpenFeign/feign) 

我的理解:feign是消费者的一种,实现了用一个伪装接口去直接调用http client端方法.

在springCloud中,和他类似的功能有ribbon.

## 二、 feign有什么用

springcloud中主要负责远程客户端的调用,如果有多个客户端那么负载均衡:轮询调用(discoveryClient).

## 三、feign怎么用

feign所有的调用都是基于接口的,所以需要调用的客户端,都需要以接口的方式做引用.

 [具体代码](https://github.com/JerryDtj/springCloud.git) [文章](http://blog.didispace.com/spring-cloud-starter-dalston-2-3/)

首先需要建立server端和client端.建立方式参考以前的博文:springCloud学习之eureka,springCloud学习之discoveryClient.本文只讲解fegin.具体代码如下:

- pom.xml

  ```
  <dependencies>
      <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-eureka</artifactId>
      </dependency>
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-web</artifactId>
      </dependency>
  
      <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-feign</artifactId>
      </dependency>
  
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-test</artifactId>
          <scope>test</scope>
      </dependency>
  </dependencies>
  ```

- Application.java

  加入注解:

  ```
  @EnableFeignClient
  @EnableDiscoveryClient
  ```

- interface

  ```
  //说明你需要调用哪个客户端
  @FeginClient("eureka-client") 
  public interface DcClient{
  	//用什么请求调用client端的哪个方法
  	@GetMapping("/dc") 
  	string consumer();
  }
  ```

- controller

  ```
  @RestController
  public class DcController{
  	//自动注入需要调用的客户端接口
  	@Autowired
  	private DcClient dcClient;
  	
  	@GetMapping("/dc")
  	public String dc(){
  		return dcClient.consumer();
  	}
  }
  ```

## 四、feign的实现原理

