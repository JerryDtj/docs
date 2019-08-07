# springCloud——consul教程

## 是什么

自从eureka停用后，springCloud推出的一个新的注册中心。

## 安装

- **window**

  - 在高级环境变量的path选项后最佳consul的路径

- **linux**

  - 把consul添加到path中

    ```
    vi /etc/profile
    export CONSUL_HOME=/usr/local/consul #路径为consul存放的位置
    export PATH=$CONSUL_HOME:$PATH
    ```

  - 执行source /etc/profile 使配置生效

## 配置

consoul 支持多种配置方式：命令行，环境变量，配置文件(json文件)。配置优先级按照列出的顺序。

想看具体的详细命令/配置参数请参考官网：[consul官网](https://www.consul.io/docs/)，[配置命令](https://www.consul.io/docs/agent/options.html)

## 启动

consul主要分为以下几种模式

- 开发者模式：consul agent -dev，不对外暴露服务端口，不能做集群，**只推荐用于单机开发使用**。
- 服务器模式：consul agent -server -bootstrap-expect 3 -data-dir /tmp/consul -node=s1 -bind=10.201.102.198 -ui-dir ./consul_ui/ -rejoin -config-dir=/etc/consul.d/ -client 0.0.0.0
  - `-server` ： 定义agent运行在server模式
  - `-bootstrap-expect` ：在一个datacenter中期望提供的server节点数目，当该值提供的时候，consul一直等到达到指定sever数目的时候才会引导整个集群，该标记不能和bootstrap共用
  - `-bind`：该地址用来在集群内部的通讯，集群内的所有节点到地址都必须是可达的，默认是0.0.0.0
  - `-node`：节点在集群中的名称，在一个集群中必须是唯一的，默认是该节点的主机名
  - `-ui-dir`： 提供存放web ui资源的路径，该目录必须是可读的
  - `-rejoin`：使consul忽略先前的离开，在再次启动后仍旧尝试加入集群中。
  - `-config-dir`：配置文件目录，里面所有以.json结尾的文件都会被加载
  - `-client`：consul服务侦听地址，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1所以不对外提供服务，如果你要对外提供服务改成0.0.0.0
- 客户端模式：consul agent -client

## 集群（主从）

1. 先对每个consul增加自己配置文件
2. 在主节点执行下面的命令：consul agent -server -bootstrap-expect=1 -data-dir=D:\consul -node=agent-one -bind=10.3.0.83 -enable-script-checks=true -config-dir=D:\consul1.5.3\web2.json -client=0.0.0.0 -ui
3. 在从节点执行下面的命令：consul agent -data-dir=D:\consul -node=agent-two -bind=10.3.0.110 -enable-script-checks=true -config-dir=D:\consul1.5.3\web.json -client=0.0.0.0 -ui

