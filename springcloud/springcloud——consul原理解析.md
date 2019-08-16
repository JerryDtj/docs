# springcloud——consul原理解析

## 调用流程概述

通过enableDiscoverClient注解对需要像consul server端注入的服务做个标记，然后调用ConsulAutoServiceRegistration的register()方法来注册具体的consul客户端。在register中，spring cloud会传递一个ConsulRegistration 对象，其中包含了服务地址，注册信息，心跳信息等。spring会根据这个对象中的类容获取到对应的需要注册的信息

## 调用详细说明

1. ### @EnableDiscoveryClient

   这个注解主要是做一个标记。如果autoRegister为true，ServiceRegistry将自动注册本地服务器。这个注解引入了一个类EnableDiscoveryClientImportSelector。主要的源代码如下：

   ```java
   @Target(ElementType.TYPE)
   @Retention(RetentionPolicy.RUNTIME)
   @Documented
   @Inherited
   @Import(EnableDiscoveryClientImportSelector.class)
   public @interface EnableDiscoveryClient {
   
   	/**
   	 * If true, the ServiceRegistry will automatically register the local server.
   	 * @return - {@code true} if you want to automatically register.
   	 */
   	boolean autoRegister() default true;
   
   }
   ```


2. ### EnableDiscoveryClientImportSelector

   这个类主要是作为一个配置文件自动注入的入口。它会引入AutoServiceRegistrationConfiguration类，开启对server端的自动注册。其部分代码如下：

   ![EnableDiscoverClient.png](http://doc.tianzijiaozi.top/EnableDiscoverClient.png)

3. ### AutoServiceRegistrationConfiguration

   这个类用了采用了门面模式，主要负责consul客户端的初始化，以及consul客户端向服务端的注册

   它被引用的地方有三处：AutoServiceRegistrationAutoConfiguration，ConsulAutoServiceRegistrationAutoConfiguration和spring-autoconfig-metadata.properties。

   它的源码如下：

   ```java
   /**
    * @author Spencer Gibb
    */
   @Configuration
   @EnableConfigurationProperties(AutoServiceRegistrationProperties.class)
   @ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
   public class AutoServiceRegistrationConfiguration {
   
   }
   ```

   ![AutoServiceRegistrationConfiguration](http://doc.tianzijiaozi.top/AutoServiceRegistrationConfiguration.png)

4. ### AutoServiceRegistrationAutoConfiguration.java

   这个类就是负责初完成注册中心的初始化，以及注册中心的一个初始化校验。

   里面引入了1个配置类，一个接口

   ![AutoServiceRegistrationAutoConfiguration](http://doc.tianzijiaozi.top/AutoServiceRegistrationAutoConfiguration.png)

   1. #### AutoServiceRegistration.java接口

      默认有一个自己的抽象类实现：AbstractAutoServiceRegistration，以及consul的实现ConsulAutoServiceRegistration。

   2. #### ConsulAutoServiceRegistration.java

      这个类是consul实现的自动注册实现类继承了AbstractAutoServiceRegistration<T>。里面主要包含注册中心的定义和注册方式。因为源码比较长，就不贴了，喜欢研究的同学可以自己去看里面consul的定义。路径为：org.springframework.cloud.consul.serviceregistry.ConsulAutoServiceRegistration.

   ![ConsulAutoServiceRegistration](http://doc.tianzijiaozi.top/ConsulAutoServiceRegistration.png)

5. ### ConsulAutoServiceRegistrationAutoConfiguration.java

   这个类主要负责consul的服务监听，注册。

   部分代码如下：

   ```java
   @Bean
   @ConditionalOnMissingBean
   public ConsulAutoRegistration consulRegistration(
   			AutoServiceRegistrationProperties autoServiceRegistrationProperties,
   			ConsulDiscoveryProperties properties, ApplicationContext applicationContext,
   			ObjectProvider<List<ConsulRegistrationCustomizer>> registrationCustomizers,
   			ObjectProvider<List<ConsulManagementRegistrationCustomizer>> managementRegistrationCustomizers,
   			HeartbeatProperties heartbeatProperties) {
   		return ConsulAutoRegistration.registration(autoServiceRegistrationProperties,
   				properties, applicationContext, registrationCustomizers.getIfAvailable(),
   				managementRegistrationCustomizers.getIfAvailable(), heartbeatProperties);
   }
   ```

   至此注册完毕。