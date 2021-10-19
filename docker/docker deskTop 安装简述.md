# docker deskTop 安装简述

## Docker Desktop for Windows 安装要求

Docker Desktop for Windows需要运行Microsoft Hyper-V。如果需要，Docker Desktop for Windows安装程序会为您启用Hyper-V，并重新启动计算机。启用Hyper-V后，VirtualBox（这是不支持Hyper-V的Windows上安装Docker Toolbox时候需要运行的虚拟机软件，这里我们默认你的Windows是支持Hyper-V的）不再起作用，但仍保留任何VirtualBox VM映像。创建的VirtualBox VM docker-machine（包括default通常在Toolbox安装期间创建的VM ）不再启动。这些VM不能与Docker Desktop for Windows一起使用。但是，您仍可以使用它docker-machine来管理远程VM。
**系统要求：**
Windows 10 64位：专业版，企业版或教育版（Build 15063或更高版本）。
在BIOS中启用虚拟化(各个主板的BIOS的操作面板不同，可咨询主板商)。通常，默认情况下启用虚拟化。
具有CPU SLAT功能。
至少4GB的运行内存。
启用Hyper-V
查看是否启用虚拟化：
按住：**Ctrl+Alt+Del** - 打开任务管理器 - 性能选项卡

Hyper-V的启用如下：

### 步骤1 - 控制面板 - 程序



## 步骤2 - 启用/关闭Windows功能



## 步骤3 - 勾选Hyper-V选项 - 确定 - 重新启动



**注意：如果您的系统不符合运行Docker Desktop for Mac的要求，则可以安装Docker Toolbox，它使用Oracle Virtual Box而不是Hyper-V。**
Doker Desktop for Windows 安装步骤
步骤1 - [下载](https://get.daocloud.io/#install-docker-for-mac-windows)



## 步骤2 - 双击打开下载程序

![img](https://img2018.cnblogs.com/blog/1739696/201907/1739696-20190728204748996-1891087991.png)

## 步骤3 - 点击OK

![img](https://img2018.cnblogs.com/blog/1739696/201907/1739696-20190728204820764-50024825.png)

## 步骤3 - 点击close

![img](https://img2018.cnblogs.com/blog/1739696/201907/1739696-20190728204925387-2115017295.png)