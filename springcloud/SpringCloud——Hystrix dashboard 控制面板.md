springcloud——Hystrix dashboard 控制面板

1.启动Hystrix 和hystrix dashboard 应用。访问http://localhost:1301/hystrix，可以看到如下界面：

![img](http://blog.didispace.com/content/images/posts/spring-cloud-starter-dalston-5-1-1.png)

- `Delay`：该参数用来控制服务器上轮询监控信息的延迟时间，默认为2000毫秒，我们可以通过配置该属性来降低客户端的网络和CPU消耗。
- `Title`：该参数对应了上图头部标题Hystrix Stream之后的内容，默认会使用具体监控实例的URL，我们可以通过配置该信息来展示更合适的标题。

2.在文本框中输入：http://hystrix-app:port/hystrix.stream 点击mo'nitor stream按钮，会进入到监控页面。

![img](http://blog.didispace.com/content/images/posts/spring-cloud-starter-dalston-5-1-2.png)

- 我们可以在监控信息的左上部分找到两个重要的图形信息：一个实心圆和一条曲线。
  - 实心圆：共有两种含义。它通过颜色的变化代表了实例的健康程度，如下图所示，它的健康度从绿色、黄色、橙色、红色递减。该实心圆除了颜色的变化之外，它的大小也会根据实例的请求流量发生变化，流量越大该实心圆就越大。所以通过该实心圆的展示，我们就可以在大量的实例中快速的发现故障实例和高压力实例。
    [![img](http://blog.didispace.com/content/images/posts/spring-cloud-starter-dalston-5-1-3.png)](http://blog.didispace.com/content/images/posts/spring-cloud-starter-dalston-5-1-3.png)
  - 曲线：用来记录2分钟内流量的相对变化，我们可以通过它来观察到流量的上升和下降趋势。
- 其他一些数量指标如下图所示：
  ![img](http://blog.didispace.com/content/images/posts/spring-cloud-starter-dalston-5-1-4.png)

