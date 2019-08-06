# spring cloud config使用与原理分析

## spring cloud config server 实现原理分析

### @EnableConfigServer

首先，查看@EnableConfigServer注解，Enable注解编程模型通常都是引入某种Configuration类来达到装配某些bean的目的。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(ConfigServerConfiguration.class)
public @interface EnableConfigServer {

}
```

### ConfigServerConfiguration

```java
@Configuration
public class ConfigServerConfiguration {
   class Marker {}

   @Bean
   public Marker enableConfigServerMarker() {
      return new Marker();
   }
}
```

ConfigServerConfiguration类里面并没有实现太多bean的装配，这里利用一种折中方式，引入需要的自动配置。请看下面的类。Marker唯一被引用的地方在ConfigServerAutoConfiguration类

### ConfigServerAutoConfiguration

```java
@Configuration
@ConditionalOnBean(ConfigServerConfiguration.Marker.class)
@EnableConfigurationProperties(ConfigServerProperties.class)
@Import({ EnvironmentRepositoryConfiguration.class, CompositeConfiguration.class, ResourceRepositoryConfiguration.class,
      ConfigServerEncryptionConfiguration.class, ConfigServerMvcConfiguration.class })
public class ConfigServerAutoConfiguration {

}
```

@ConditionalOnBean(ConfigServerConfiguration.Marker.class)表示当装配了ConfigServerConfiguration.Marker的实例时才会执行ConfigServerAutoConfiguration的处理。这里又另外引入了5个配置类。分析config server，重点看EnvironmentRepositoryConfiguration类。

### EnvironmentRepositoryConfiguration

```java
@Configuration
@EnableConfigurationProperties({ SvnKitEnvironmentProperties.class,
      JdbcEnvironmentProperties.class, NativeEnvironmentProperties.class, VaultEnvironmentProperties.class })
@Import({ CompositeRepositoryConfiguration.class, JdbcRepositoryConfiguration.class, VaultRepositoryConfiguration.class,
      SvnRepositoryConfiguration.class, NativeRepositoryConfiguration.class, GitRepositoryConfiguration.class,
      DefaultRepositoryConfiguration.class })
public class EnvironmentRepositoryConfiguration {
}
```

这里的@Import又引入了7种配置类，查看文档会发现其实刚好对应config server的几种实现方式git的实现方式使用的配置类就是GitRepositoryConfiguration。以GitRepositoryConfiguration的为例分析。

### GitRepositoryConfiguration
```java
@Configuration
@Profile("git")
class GitRepositoryConfiguration extends DefaultRepositoryConfiguration {
}
```

可以看出，GitRepositoryConfiguration其实是默认的实现方式，查看DefaultRepositoryConfiguration的代码。

```java
@Configuration
@ConditionalOnMissingBean(value = EnvironmentRepository.class, search = SearchStrategy.CURRENT)
class DefaultRepositoryConfiguration {
   @Autowired
   private ConfigurableEnvironment environment;

   @Autowired
   private ConfigServerProperties server;

   @Autowired(required = false)
   private TransportConfigCallback customTransportConfigCallback;

   @Bean
   public MultipleJGitEnvironmentRepository defaultEnvironmentRepository(
           MultipleJGitEnvironmentRepositoryFactory gitEnvironmentRepositoryFactory,
         MultipleJGitEnvironmentProperties environmentProperties) throws Exception {
      return gitEnvironmentRepositoryFactory.build(environmentProperties);
   }
}
```

最终是装配一个MultipleJGitEnvironmentRepository的bean，实际每种配置类的实现的最终都是装配一个EnvironmentRepository的子类，可以认为，有一个地方最终会引用到EnvironmentRepository的bean，使用org.springframework.cloud.config.server.environment.EnvironmentRepository#findOne方法来查询配置。

### EnvironmentController

尝试搜索使用到findOne方法的类，org.springframework.cloud.config.server.environment.EnvironmentController#labelled中使用到，而且这里面是创建了一个RestController，推测应该是客户端获取服务端配置的入口，查看代码如下。

```java
@RequestMapping("/{name}/{profiles}/{label:.*}")
public Environment labelled(@PathVariable String name, @PathVariable String profiles,
      @PathVariable String label) {
   if (name != null && name.contains("(_)")) {
      // "(_)" is uncommon in a git repo name, but "/" cannot be matched
      // by Spring MVC
      name = name.replace("(_)", "/");
   }
   if (label != null && label.contains("(_)")) {
      // "(_)" is uncommon in a git branch name, but "/" cannot be matched
      // by Spring MVC
      label = label.replace("(_)", "/");
   }
   Environment environment = this.repository.findOne(name, profiles, label);
   if(!acceptEmpty && (environment == null || environment.getPropertySources().isEmpty())){
       throw new EnvironmentNotFoundException("Profile Not found");
   }
   return environment;
}
```

注意这里的EnvironmentController#repository属性就是GitRepositoryConfiguration实例化的MultipleJGitEnvironmentRepository,如果是别的实现方式就是别的**EnvironmentRepository**。可以看出”/{name}/{profiles}/{label:.*}”路径参数正好与我们的请求方式相对应，因此Config Server是通过建立一个RestController来接收读取配置请求的，然后使用**EnvironmentRepository**来进行配置查询，返回一个org.springframework.cloud.config.environment.Environment对象的json串，推测客户端接收时也应该是反序列化为org.springframework.cloud.config.environment.Environment的一个实例。可以看一下Environment的属性定义。

```java
private String name;

private String[] profiles = new String[0];

private String label;

private List<PropertySource> propertySources = new ArrayList<>();

private String version;

private String state;
```

### 尝试自定义EnvironmentRepository 实现

在上面的分析可以知道，所有的配置EnvironmentRepository的Configuration都是在没有EnvironmentRepository的bean的时候才会生效，我们可以实现自定义的EnvironmentRepository的bean，然后就可以覆盖的系统的实现。代码如下。

```java
private Environment getRemoteEnvironment(RestTemplate restTemplate,
      ConfigClientProperties properties, String label, String state) {
   String path = "/{name}/{profile}";
   String name = properties.getName();
   String profile = properties.getProfile();
   String token = properties.getToken();
   int noOfUrls = properties.getUri().length;
   if (noOfUrls > 1) {
      logger.info("Multiple Config Server Urls found listed.");
   }

   Object[] args = new String[] { name, profile };
   if (StringUtils.hasText(label)) {
      if (label.contains("/")) {
         label = label.replace("/", "(_)");
      }
      args = new String[] { name, profile, label };
      path = path + "/{label}";
   }
   ResponseEntity<Environment> response = null;

   for (int i = 0; i < noOfUrls; i++) {
      Credentials credentials = properties.getCredentials(i);
      String uri = credentials.getUri();
      String username = credentials.getUsername();
      String password = credentials.getPassword();

      logger.info("Fetching config from server at : " + uri);

      try {
         HttpHeaders headers = new HttpHeaders();
         addAuthorizationToken(properties, headers, username, password);
         if (StringUtils.hasText(token)) {
            headers.add(TOKEN_HEADER, token);
         }
         if (StringUtils.hasText(state) && properties.isSendState()) {
            headers.add(STATE_HEADER, state);
         }

         final HttpEntity<Void> entity = new HttpEntity<>((Void) null, headers);
         response = restTemplate.exchange(uri + path, HttpMethod.GET, entity,
               Environment.class, args);
      }
      catch (HttpClientErrorException e) {
         if (e.getStatusCode() != HttpStatus.NOT_FOUND) {
            throw e;
         }
      }
      catch (ResourceAccessException e) {
         logger.info("Connect Timeout Exception on Url - " + uri
               + ". Will be trying the next url if available");
         if (i == noOfUrls - 1)
            throw e;
         else
            continue;
      }

      if (response == null || response.getStatusCode() != HttpStatus.OK) {
         return null;
      }

      Environment result = response.getBody();
      return result;
   }

   return null;
}
```

上面的代码主要操作就是拼接一个请求配置地址串，获取所需的ApplicationName，profile，label参数，利用RestTemplate执行http请求，返回的json反序列化为Environment，从而获得所需要的配置信息。

那么问题来了，client是在什么时候调用getRemoteEnvironment方法的，推测应该是在boostrap context进行初始化阶段。在getRemoteEnvironment打个断点，重启client程序，可以查看到以下调用链路。

org.springframework.boot.SpringApplication#run(java.lang.String…)

org.springframework.boot.SpringApplication#prepareContext

org.springframework.boot.SpringApplication#applyInitializers

org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration#initialize

org.springframework.cloud.config.client.ConfigServicePropertySourceLocator#locate

org.springframework.cloud.config.client.ConfigServicePropertySourceLocator#getRemoteEnvironment

所以，可以知道在spring启动时就会远程加载配置信息，SpringApplication#applyInitializers代码如下，会遍历所有initializer进行一遍操作，PropertySourceBootstrapConfiguration就是其中之一的initializer。

```java
protected void applyInitializers(ConfigurableApplicationContext context) {
   for (ApplicationContextInitializer initializer : getInitializers()) {
      Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
            initializer.getClass(), ApplicationContextInitializer.class);
      Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
      initializer.initialize(context);
   }
}
```

当引入了spring-cloud-config后PropertySourceBootstrapConfiguration#propertySourceLocators中会新增一个ConfigServicePropertySourceLocator实例。在PropertySourceBootstrapConfiguration#initialize中遍历propertySourceLocators的locate方法，然后读取远程服务配置信息；如果没有引入了spring-cloud-config，那么propertySourceLocators将会是空集合。代码如下。

```java
@Override
public void initialize(ConfigurableApplicationContext applicationContext) {
   CompositePropertySource composite = new CompositePropertySource(
         BOOTSTRAP_PROPERTY_SOURCE_NAME);
   AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
   boolean empty = true;
   ConfigurableEnvironment environment = applicationContext.getEnvironment();
   for (PropertySourceLocator locator : this.propertySourceLocators) {
      PropertySource<?> source = null;
      source = locator.locate(environment);
      if (source == null) {
         continue;
      }
      logger.info("Located property source: " + source);
      composite.addPropertySource(source);
      empty = false;
   }
   if (!empty) {
      MutablePropertySources propertySources = environment.getPropertySources();
      String logConfig = environment.resolvePlaceholders("${logging.config:}");
      LogFile logFile = LogFile.get(environment);
      if (propertySources.contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
         propertySources.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
      }
      insertPropertySources(propertySources, composite);
      reinitializeLoggingSystem(environment, logConfig, logFile);
      setLogLevels(applicationContext, environment);
      handleIncludedProfiles(environment);
   }
}
```

### PropertySourceBootstrapConfiguration#propertySourceLocators初始化

`@Autowired(required = false)`
`private List<PropertySourceLocator> propertySourceLocators = new ArrayList<>();`
上面的代码可以看出，这里的propertySourceLocators是直接注入上下文中管理的PropertySourceLocator实例，所以PropertySourceLocator一定有别的地方初始化。

搜索ConfigServicePropertySourceLocator的使用处，发现org.springframework.cloud.config.client.ConfigServiceBootstrapConfiguration#configServicePropertySource方法装配了一个ConfigServicePropertySourceLocator的bean，代码如下。

```java
@Configuration
@EnableConfigurationProperties
public class ConfigServiceBootstrapConfiguration {

@Bean
@ConditionalOnMissingBean(ConfigServicePropertySourceLocator.class)
@ConditionalOnProperty(value = "spring.cloud.config.enabled", matchIfMissing = true)
public ConfigServicePropertySourceLocator configServicePropertySource(ConfigClientProperties properties) {
   ConfigServicePropertySourceLocator locator = new ConfigServicePropertySourceLocator(
         properties);
   return locator;
}
   //........ 
}
```

org.springframework.cloud.config.client.ConfigServiceBootstrapConfiguration是config client的类，当引入了spring cloud config时引入，再尝试搜索使用处，发现在spring cloud config client包里面的spring.factories里面引入了ConfigServiceBootstrapConfiguration，熟悉spring boot自动装配的都知道，程序会自动加载spring.factories里面的配置类。

**也就是说，当引入了spring cloud config client包，就会自动加载ConfigServiceBootstrapConfiguration类，自动装配ConfigServiceBootstrapConfiguration里面配置的bean，也就自动实例化一个ConfigServicePropertySourceLocator。**

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.config.client.ConfigClientAutoConfiguration

# Bootstrap components
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.config.client.ConfigServiceBootstrapConfiguration,\
org.springframework.cloud.config.client.DiscoveryClientConfigServiceBootstrapConfiguration
```

