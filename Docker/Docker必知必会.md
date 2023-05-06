# Docker必知必会

## 概念

Docker是一种轻量级的虚拟化技术，可以让开发者非常便捷地实现应用程序的打包、移植、启动等操作，在软件开发、交付和部署中，有非常广泛的应用。

Docker容器与传统虚拟机的架构对比如下：

<img src="file:///C:/Users/YF/Desktop/dss/workspace/step_by_step/Docker/resources/vm-vs-docker.jpg" title="" alt="Docker VS VM" style="zoom:50%;">

传统VM使用Hypervisor通过对物理主机的硬件资源进行协调和管理，为每个GuestOS分配独立的资源，让每个GuestOS成为一个虚拟主机，不同的GuestOS中的应用程序互不影响。
Docker容器直接运行在物理主机的HostOS上，共用物理主机的硬件资源，Container Engine负责实现容器之间的资源隔离，让每个容器的应用独立地运行。
可以看出，容器比虚拟机少了一层GuestOS，容器占用资源更少，启动更快，但隔离程度不如虚拟机。容器和虚拟机简要对比如下：

| 维度   | 容器   | 虚拟机   |
| ---- | ---- | ----- |
| 隔离级别 | 进程级  | 操作系统级 |
| 镜像大小 | K-M级 | M-G级  |
| 启动时间 | 秒级   | 分钟级   |
| 性能表现 | 接近原生 | 有损耗   |
| 移植性  | 轻量   | 重量    |
| 隔离性  | 弱    | 强     |
| 安全性  | 弱    | 强     |

Docker中有三个常见的名词，这里先简单介绍下概念，知道是什么就行，后面再详细说明。

### 镜像（Image）

镜像是一个特殊的文件系统，提供容器运行时所需的环境和配置，例如程序、库、资源、配置等文件，以及环境变量、匿名卷、用户等配置参数。镜像是静态的，不包含任何动态数据，在镜像构建之后，其内容不会发生改变。

### 容器（Container）

容器和镜像的关系，类似于面向对象编程中对象和类的关系，容器是运行镜像后得到的实例，运行镜像就相当于类的实例化，多次运行镜像，可以得到多个容器。容器是动态的，可以对容器进行创建、删除、启动、停止、暂停等操作。
容器实质上是运行在宿主机上的进程，Docker是用特殊的技术将容器与宿主机上的其他进程隔离开来，使得容器内的应用看起来是运行在一个独立的环境中。

### 仓库（Repository）

仓库类似github，对镜像进行存储和分发。在任一宿主机上，都可以从仓库拉取指定镜像，也可以把自己打包好的镜像上传到仓库，供他人访问。默认的是官方仓库Docker Hub，拥有众多官方镜像，国内访问需要配加速器，如阿里云的镜像仓。也可以自行搭建本地私有镜像仓。

## 原理

前面提到，容器是宿主机用特殊机制隔离出来的进程。为了实现容器进程的互不干扰，这个机制需要解决两个基本问题：

1. 容器内屏蔽容器外的情况，使用Linux的Namespace机制
2. 容器拥有独立的资源，使用Linux的Cgroups机制

### Namespace

顾名思义，Namespace就是命名空间。C++使用命名空间解决了类型、变量和函数的冲突问题。Docker容器也具有自己的命名空间，通过命名空间对User、Pid、Mount、UTS、IPC、Network进行隔离，使得不同的容器进程号、用户、文件目录等相互屏蔽。

### Cgroups

Linux Cgroups的全称是Linux Control Group，主要用于对共享资源进行隔离、限制、审计。通过Cgroups限制容器能够使用的资源上限，包括CPU、内存、磁盘、网络带宽等，可以避免多个容器之间的资源竞争。Linux一切皆文件，Cgroups也是通过树状的文件系统来对资源进行限制。

- 查看cgroup挂载的目录，可以看到cgroup挂在sys/fs/cgroup节点，该路径下还有很多子目录（又称子系统），如cpu、memory等，每个子系统对应一种可以被限制的资源类型。
  ![cgroup挂载](C:\Users\YF\Desktop\dss\workspace\step_by_step\Docker\resources\cgroup挂载.png)

- 以cpu为例，查看cpu子系统。其中有两个参数cfs_period_us和cfs_quota_us通常组合使用，用于限制进程在长度为cfs_period_us的时间内，只能被分配到总量为cfs_quota_us的CPU时间。还有一个tasks文件，其中存放的是受限制的进程编号。
  ![cpu子系统](C:\Users\YF\Desktop\dss\workspace\step_by_step\Docker\resources\cpu子系统.png)
  ![cpu限制](C:\Users\YF\Desktop\dss\workspace\step_by_step\Docker\resources\cpu限制.png)  

- cpu子系统中有个docker目录，docker目录中的文件与cpu目录中的文件一样。当我们拉起一个容器，比如redis，可以看到docker目录中又多了一层以容器id为名称的目录。
  ![docker子目录](C:\Users\YF\Desktop\dss\workspace\step_by_step\Docker\resources\docker子目录.png)
  ![容器子目录](C:\Users\YF\Desktop\dss\workspace\step_by_step\Docker\resources\容器子目录.png)  
  
  综上两点，容器其实是一个启用了多种Namespace的进程，它能够使用的资源量收到Cgroups的限制。

## 使用








