# idea 和 docker 联合使用

### 一、在你启动好的docker任务栏图标上，右击settings选项

![image-20200618150145145](C:%5CUsers%5Cpc%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20200618150145145.png)

## 二、Idea配置

1、确保Idea安装了docker插件

 

![img](https://img2018.cnblogs.com/blog/152140/201906/152140-20190619102931165-16976042.png)

2、在项目根目录下新建Dockerfile,配置如下

```dockerfile
#指定基础镜像，在其上进行定制
FROM java:8
#指定作者
MAINTAINER jerry
#指定标签
LABEL name="mw-eureka" version="1.0" author="jerry"
#这里的 /home/app/work/eureka 目录就会在运行时自动挂载为匿名卷，任何向 /data 中写入的信息都不会记录进容器存储层
VOLUME /home/app/work/eureka
#复制上下文目录下的target/xlc-cloud-eureka-0.0.1.jar 到容器里
ADD  target/xlc-cloud-eureka-0.0.1.jar eureka.jar
#声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务
EXPOSE 3000
#指定容器启动程序及参数   <ENTRYPOINT> "<CMD>"
ENTRYPOINT ["java","-jar","eureka.jar"]
```

3、配置Docker服务器

![image-20200618151149106](pic/image-20200618151149106.png)

4、配置docker发布

![docker3](pic/docker3.png)

5、打包

![img](https://img2018.cnblogs.com/blog/152140/201906/152140-20190619103323446-422991513.png)

5、一键部署

(1)打开Dokcer窗口

![img](https://img2018.cnblogs.com/blog/152140/201906/152140-20190619103502673-1151938467.png)

(2)部署

![image-20200618152907907](pic/docker4.png)

(3)发布完成

![image-20200618153113551](pic/docker5.png)

至此，idea中的项目就已经发布到docker中了，然后就可以本地直接访问了。