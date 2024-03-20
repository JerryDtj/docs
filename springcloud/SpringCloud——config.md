# Spring Cloud——Config

## 是什么

项目微服务化后，由于每个微服务都有自己的一个配置文件，导致了配置文件繁多，修改每一个微服务的配置都会修改所有的横向配置。所以诞生了这么一个配合中心。

它可以把所有的配置集中管理，达到修改配置只需要修改单个配置文件的目的。

## 怎么用

[代码下载(config-client,config-server,config-repo)](https://github.com/JerryDtj/springCloud)

1. 新建一个Spring Boot项目（config-server）作为配置中心的服务端

   - pom.xml文件添加如下依赖

     ```xml
     <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-config-server</artifactId>
     </dependency>
     ```

   - application.yml内容如下

     ```yaml
     server:
       port: 10080
     spring:
       application:
     #    项目地址
         name: config-server
       cloud:
         config:
           server:
     #        对应的git地址
             git:
               uri: https://github.com/JerryDtj/springCloud.git
     #          查找目录，如果server项目为一个子模块，这个可以不要
               search-paths: config-repo
     ```

   - 启动类，具体实现略过，但是需要添加**@EnableConfigServer**注解来开启springCloud的配置中心

2. 新建一个Spring Boot项目作为配置中心的客户端（config-client）

   - pom.xml:注意因为springboot 版本不同，添加的文件不一致

     ```xml
     <!--        如果springboot版本为2.x，那么需要加入这个依赖，否则无法通过http的方式访问到配置中心的资源
             <dependency>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-starter-actuator</artifactId>
             </dependency>-->
             <dependency>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-starter-web</artifactId>
             </dependency>
             <dependency>
                 <groupId>org.springframework.cloud</groupId>
                 <artifactId>spring-cloud-starter-config</artifactId>
             </dependency>
     ```

   - bootstrap.yml

     ```yml
     server:
       port: 10081
     spring:
       application:
     #    服务名
         name: config-client
       cloud:
         config:
     #      配置中心服务器地址
           uri: http://localhost:10080/
     #      加载哪个配置文件,默认default
           profile: dev
     #      git的分支文件
           label: master
           retry:
             # 配置重试次数，默认为6
             max-attempts: 6
             # 间隔乘数 默认1.1
             multiplier: 1.1
             # 初始重试间隔时间，默认1000ms
             initial-interval: 1000
             # 最大间隔时间，默认2000ms
             max-interval: 2000
     ```

   - 启动类，这里直接略过，正常的启动类即可

   - 访问controller

     ```java
     package com.learn.configclient.contorller;
     
     import org.springframework.beans.factory.annotation.Value;
     import org.springframework.web.bind.annotation.GetMapping;
     import org.springframework.web.bind.annotation.RequestMapping;
     import org.springframework.web.bind.annotation.RestController;
     
     /**
      * despaction
      *
      * @Author: jerry
      * @date: 2019/8/5 16:39
      * @description:
      **/
     @RestController
     @RequestMapping("/yml")
     public class YmlController {
     
         @Value("${info.add}")
         private String add;
     
         @GetMapping("/add")
         public String info(){
             System.out.println(add);
             return add;
         }
     
     }
     ```

3. 添加客户端（config-client）的配置文件

   - **如果配置中心作为一个单独项目**

     直接在配置中心项目（config-server）的resource下面新建配置文件

     - config-client.yml

       ```yaml
       info:
         profile: default
         add: test
       ```

     - config-client-dev.xml

       ```yaml
       info:
         profile: dev1
         add: test1
       ```

   - **如果配置中心作为一个maven子项目并且所有的配置文件为父项目根目录下的指定文件夹**（config-repo）

     在父项目的pom文件夹同级目录添加上面的两个配置文件即可

4. 访问

   - 客户端项目访问：直接打开浏览器访问即可

   - web项目访问

     - 客户端springboot版本为2.x：路径为http://localhost:10080/config-client.yml或者http://localhost:2001/actuator/config-client-dev/default

     - 客户端springboot版本为1.x：路径为http://localhost:10080/config-client.yml或者http://localhost:10080/config-client-dev/default

       这里有个疑问：按理来说，路径格式为：http://url/项目名/配置文件名，但是这里不知道为什么项目名直接为配置文件名。然后后面的配置文件名无效。才会显示出正确的路径。可能具体的原因要看过源码后才知道。