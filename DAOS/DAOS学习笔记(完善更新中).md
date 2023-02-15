# 概述 
    该笔记主要为个人对 Intel DASO官方英文文档(中的第2，3，8，11，12，13章节) 以及 相关论坛DAOS 文章的学习心得和记录，作为个人学习成果输出，仅仅用于学习和交流使用。 个人理解和能力有限，难免会有错误和理解不到位的情况，也难免有疏忽，欢迎积极交流，沟通和纠错。

 # DAOS
   ## 什么是DAOS?
    DAOS： Distributed Asychronous Object Storage  分布式异步对象存储 (属于高性能的分布式存储系统)。
   可以简单将DAOS理解为软件栈 + 硬件栈 对象存储容器。
   (<软件(PMDK+SPDK) + 硬件(nvme)>)

   看文档说明，应该是和 k-v数据库很像（本身对象存储就是KV存储）。只不过DAOS 是软硬件一体，intel 提供DAOS 软件（api）配合 nvme和ram等硬件，合体成DAOS，变成了一个kv分布式数据库。

   分布式异步对象存储 (DAOS) 是一个开源的对象存储系统，专为大规模分布式非易失性内存 (NVM, Non-Volatile Memory) 设计，利用了 SCM(Storage-Class Memory，比如intel的傲腾) 和 NVMe(Non-Volatile Memory express) 等的下一代 NVM 技术。

   DAOS 服务器将其元数据保存在持久内存中，而将数据保存在 NVMe 固态盘中。

# 为什么要用DAOS
   上一代的分布式存储是针对机械硬盘设计和优化的，现在固态硬盘已经大量使用，上一代分布式存储软件已经不适应。需要新一代分布式存储软件--比如：DAOS。

# 介绍DAOS(分布式异步对象存储)的架构组件构成
## 相关组件：
   DAOS 的安装涉及多个组件，这些组件可以是集中式的，也可以是分布式的。

   DAOS 软件定义存储 (software-defined storage, SDS) 框架依赖于两种不同的通信通道：
      1.用于带外管理 (out-of-band management) 的 TCP/IP 网络；
      2.用于数据访问的高性能结构。


   DAOS 服务器是一个多租户守护进程，运行在物理节点、VM 或容器上，管理分配给 DAOS 本地连接的 SCM (Storage-Class Memory) 和 NVM (Non-Volatile Memory) 存储。它监听由 IP 地址和 TCP 端口号寻址的管理端口，以及由网络 URI 寻址的一个或多个结构端点。

   DAOS 服务器是通过 YAML 文件 /etc/daos/daos_server.yml（可通过命令行指定的其他路径）进行配置的。服务的启动和停止可以与不同的守护进程管理或编排框架集成（systemd 脚本、Kubernetes 服务、或类似 pdsh 和 srun 的并行启动程序）。

   DAOS 系统由一个系统名标识，它由一组连接到同一结构的 DAOS 服务器组成。两个不同的系统由两组不相交的服务器组成，彼此不能相互协调。DAOS Pool 也不能跨多个系统。

   在内部，DAOS 服务器由多个守护进程组成：
   首先要启动的是**控制平面**（control plane，二进制名 daos_server）。
      它负责解析配置文件、配置存储并最终启动和监视数据平面的一个或多个实例。
      控制平面用 Go 编写，并在 gRPC 框架上实现 DAOS management API。该框架提供了一个安全的带外通道来管理 DAOS 系统。
      每个服务器要启动的数据平面实例的数量以及存储、CPU 和 Fabric Interface Affinity 可以通过 daos_server.yml 进行配置。

   然后是**数据平面**（data plane，二进制名 daos_engine）。
      数据平面是一个用 C 编写的多线程进程，负责运行 DAOS 存储引擎。它通过 CART 通信中间件处理传入的元数据和 I/O 请求，并通过 PMDK（Persistent Memory Devevelopment Kit，用于 SCM）和 SPDK（Storage Performance Development Kit，用于 NVMe SSD）访问本地 NVM 存储。
      数据平面依赖于 ABT (Argobots) 进行基于事件的并行处理，并导出可通过结构独立寻址的多个 Target。
      在 DAOS 系统中，每个数据平面实例都被分配一个唯一的等级。

   控制平面和数据平面进程通过 Unix Domain Sockets 和被称为 dRPC 的定制轻量级协议进行本地通信。

## DAOS 分布式异步对象存储｜控制平面
   







## DAOS 分布式异步对象存储｜数据平面 




      

