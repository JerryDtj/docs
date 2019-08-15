# springcloud——consul原理解析

## 调用流程概述

通过enableDiscoverClient注解对需要像consul server端注入的服务做个标记，然后调用ConsulAutoServiceRegistration的register()方法来注册具体的consul客户端。在register中，spring cloud会传递一个ConsulRegistration 对象，其中包含了服务地址，注册信息，心跳信息等。spring会根据这个对象中的类容获取到对应的需要注册的信息

## 调用详细说明

