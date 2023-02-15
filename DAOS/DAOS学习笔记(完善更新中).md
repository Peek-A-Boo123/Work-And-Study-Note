概述 
    该笔记主要为个人对 Intel DASO官方英文文档(中的第2，3，8，11，12，13章节) 以及 相关论坛DAOS 文章的学习心得和记录，作为个人学习成果输出，仅仅用于学习和交流使用。 个人理解和能力有限，难免会有错误和理解不到位的情况，也难免有疏忽，欢迎积极交流，沟通和纠错。

 DAOS
    什么是DAOS?
    DAOS： Distributed Asychronous Object Storage  分布式异步对象存储 (属于高性能的分布式存储系统)。
   可以简单将DAOS理解为软件栈 + 硬件栈 对象存储容器。
   (<软件(PMDK+SPDK) + 硬件(nvme)>)

   看文档说明，应该是和 k-v数据库很像（本身对象存储就是KV存储）。只不过DAOS 是软硬件一体，intel 提供DAOS 软件（api）配合 nvme和ram等硬件，合体成DAOS，变成了一个kv分布式数据库。

   分布式异步对象存储 (DAOS) 是一个开源的对象存储系统，专为大规模分布式非易失性内存 (NVM, Non-Volatile Memory) 设计，利用了 SCM(Storage-Class Memory，比如intel的傲腾) 和 NVMe(Non-Volatile Memory express) 等的下一代 NVM 技术。

   DAOS 服务器将其元数据保存在持久内存中，而将数据保存在 NVMe 固态盘中。

为什么要用DAOS
   上一代的分布式存储是针对机械硬盘设计和优化的，现在固态硬盘已经大量使用，上一代分布式存储软件已经不适应。需要新一代分布式存储软件--比如：DAOS。

介绍DAOS(分布式异步对象存储)的架构组件构成
相关组件：
   DAOS 的安装涉及多个组件，这些组件可以是集中式的，也可以是分布式的。

   DAOS 软件定义存储 (software-defined storage, SDS) 框架依赖于两种不同的通信通道：
      1.用于带外管理 (out-of-band management) 的 TCP/IP 网络；
      2.用于数据访问的高性能结构。


      DAOS 服务器是一个多租户守护进程，运行在物理节点、VM 或容器上，管理分配给 DAOS 本地连接的 SCM (Storage-Class Memory) 和 NVM (Non-Volatile Memory) 存储。它监听由 IP 地址和 TCP 端口号寻址的管理端口，以及由网络 URI 寻址的一个或多个结构端点。



      

