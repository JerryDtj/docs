# Docker DeskTop 简介

相信大家已经对微服务有一定认识了，不知道大家在日常工作中是否碰到过这样的情景。比如说，

​	我现在属于回归组，我对于ums 是不怎么关心的。但是呢，我的joyhr里面的一些服务还是依赖于ums的一些服务接口。

​	我现在是在微服务拆分组，我对joyhr 是不怎么关心得，但是我要迁移的一些服务又依赖于joyhr的一些服务接口。

​	现在每次启动都要启动eureka，我比较懒，这个有没有什么办法不启动啊，每次都要启动太烦了。

为了解决这些问题，当项目相对稳定后，docker技术就孕育而生了。

## Docker

Docker是一个为开发者和运维工程师（系统管理员）以容器的方式构建，分享和运行应用的平台．使用容器进行应用部署的方式，我们成为容器化．

容器化应用具有一下特性，使得容器化日益流行：

- 灵活：　再复杂的应用都可以进行容器化．
- 轻量：　容器使用使用和共享主机的内核，在系统资源的利用比虚拟机更加高效．
- 可移植：　容器可以本地构建，部署到云上，运行在任何地方．
- 松耦合：　容器是高度自封装的，可以在不影响其他容器的情况下替换和升级容器．
- 可扩展：　可以在整个数据中心里增加和自动分发容器副本．
- 安全：　容器约束和隔离应用进程，而无需用户进行任何配置．

### 镜像和容器

其实，容器就是运行的进程，附带一些封装的特性，使其与主机上和其他容器的进程隔离．每个容器都只访问它自己私有的文件系统，这是容器隔离很重要的一方面．而Docker镜像就提供了这个文件系统，一个镜像包含运行该应用所有需求 - 代码或者二进制文件，运行时，依赖库，以及其他需要的文件系统对象．

通过与虚拟机对比，虚拟机（VM）通过一个虚拟机管理（Hypervisor）运行了完整的操作系统来访问主机资源．通常虚拟机会产生大量的开销，超过了应用本身所需要的开销．

![Container-VM.png](https://segmentfault.com/img/bVbDpes)

### 容器编排

容器化过程的可移植性和可重复性意味着我们有机会跨云和数据中心移动和扩展容器化的应用程序，容器有效地保证应用程序可以在任何地方以相同的方式运行，这使我们可以快速有效地利用所有这些环境．当我们扩展我们的应用，我们需要一些工具来帮助自动维护这些应用，在容器的生命周期里，可以自动替换失败的容器，管理滚动升级，以及重新配置．　　

容器编排器（Orchestrator）就是管理，扩展和维护容器化应用的工具，当前最常见的例子就是 *Kubernetes* 和 *Docke Swarm* ．_Docker Desktop_ 工具可以在开发环境提供这两个编排工具．

