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
      新存储介质：SSD （NVMe协议）
      存储引擎：高性能LSM存储引擎

# 介绍DAOS(分布式异步对象存储)的组件架构
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

   **控制平面**和**数据平面**进程通过 Unix Domain Sockets 和被称为 dRPC 的定制轻量级协议进行本地通信。

## DAOS存储模型（Storage Model） 
   DAOS 存储模型的基本抽象概括:  同一个文件夹附件中的 "DAOS存储模型" 图。

   DAOS Pool 是 跨Target 集合分布的的预留存储资源。分配给每个 Target 上持有的 Pool 的实际空间称为池分片 （Pool Shard）。

   分配给 Pool 的总空间在创建时确定，后期可以通过调整所有 Pool Shard 的大小（不超过 Target 专用的存储容量限制）或添加更多 Target（添加更多 Pool Shard）来扩展它。

   池提供了存储虚拟化，（池）是资源调配和隔离的单元。DAOS Pool 不能跨多个系统。

   一个 池 Pool 可以承载多个称为 DAOS Container 的事务对象存储。每个 Container 都是一个私有的对象地址空间，可以对其进行事务性修改，并且独立于在同一 Pool 中的其他 Container。Container 是快照和数据管理的单元。一个 Container 内的 DAOS 对象可以分布在当前 Pool 的任何一个 Target 上以提高性能和恢复能力，并且可以通过不同的 API 访问，从而高效地表示结构化、半结构化和非结构化数据。

### DAOS Pool
   池由唯一的池 UUID 标识，并在 pool map 中维护 Target 信息（memberships 成员身份）。memberships 是确定的和一致的，memberships 的变更是按顺序编号的。 pool map 不仅记录活跃 Target 的列表，还以树的形式包含存储拓扑，用于记录（标记） 共享公共硬件组件 的 Target。例如，树的第一级可以表示共享同一主板的 Target，第二级可以表示共享同一机架的所有主板，最后第三级可以表示同一机房中的所有机架。

   该框架有效地表示了层次化的容错域，然后使用这些容错域来避免将冗余（备份）数据放置在发生相关故障的 Target 上。在任何时候，都可以将新 Target 添加到 pool map 中，并且可以排除失败的 Target。此外，pool map 是完全版本化的，这有效地为map的每次修改分配了唯一的序列，特别是对于失败节点的删除。

   池分片（Pool Shard） 是永久内存的预留空间，可以选择与特定 Target 上 NVMe 预先分配的空间相结合。它有一个固定的容量，并且在满了就操作失败。可以随时查询当前空间使用情况，并报告 池分片（Pool Shard） 中存储的任何数据类型所使用的总字节数。

   在target失败并从pool map 中排除时，池内的冗余数据（副本数据）会自动在线恢复。这种自我修复过程称为重建。
   重建进度定期记录在 持久内存中 的存储池中的特殊日志中，用于定位级联故障。
   添加新target时，数据会自动迁移到新添加的target，以在所有成员之间平均重新分配空间使用量。此过程称为空间重新平衡，并使用专用的持久日志来支持中断和重新启动。

   创建 Pool 时，必须定义一组系统属性以配置 Pool 支持的不同功能。此外，用户还可以定义将持久存储的属性。
   只有经过身份验证和授权的应用程序才能访问池。 可以支持多种安全框架。

   
   小结：池是分布在不同存储节点上的一组target，数据和元数据分布在这些节点上以实现水平可扩展性，并进行复制或EC编码（纠删码 ，erasure code)以确保持久性和可用性。


### DAOS Container
   容器表示一个池内的对象地址空间，由容器 UUID 标识。 

   与 Pool 一样，Container 可以存储用户属性。Container 在创建时必须传递一组属性，以配置不同的功能，例如校验和。

   要访问 Container，应用程序必须首先连接到 Pool，然后打开 Container。如果应用程序被授权访问 Container，则返回 Container 句柄，它的功能包括授权应用程序中的任何进程访问 Container 及其内容。打开 Container的进程可以与其任何或所有peer共享该句柄。它们的功能在 Container 关闭时被撤销。

   Container 中的对象可能具有不同的schema（既数据组织形式），用于应对 数据分布存储和 （管理）Target 上的冗余（副本）， 动态或静态条带化、复制或EC（纠删码）是 定义对象schema（既数据组织形式）所需的一些属性。

   Container 是事务和版本控制的基本单元。

   所有的对象操作都被 DAOS 库 用一个称为 epoch 的时间戳隐式地标记。DAOS 事务 API 允许组合多个对象更新到单个原子事务中，并基于 epoch 顺序进行多版本并发控制。所有版本更新都可以定期聚合，以回收重叠写入所占用的空间，并降低元数据复杂性。快照是一个永久引用，可以放置在特定的 epoch 上以防止聚合。

   Container 元数据（快照列表、打开的句柄、对象类、用户属性、属性和其他）存储在持久性内存中，并由专用 Container 元数据服务维护，该服务使用与父元数据 Pool 服务相同的复制引擎或自己的引擎，这在创建 Container 时是可配置的。

   与 Pool 一样，对 Container 的访问由 Container 句柄控制。要获取有效的句柄，应用程序进程必须打开 Container 并通过安全检查。

### DAOS Object
   为了避免传统存储系统常见的扩展问题和开销，DAOS 对象有意简化。 
   没有提供超出type 和 schema 的默认对象元数据。 这意味着系统不维护时间、大小、所有者、权限甚至跟踪开启者。

   为了实现高可用性和水平可伸缩性，提供了许多对象schema（复制/EC、静态/动态条带化等）。 schema框架灵活且易于扩展，以允许将来自定义新的schema类型。 分布是根据对象标识符和pool map在对象打开时通过算法生成的。 通过在网络传输和存储期间使用校验和保护对象数据来确保端到端完整性。

   可以通过不同的 API 访问 DAOS 对象：
      Multi-level key-array API 是具有局部性特征的本机对象接口。密钥分为分发密钥 (dkey) 和属性密钥 (akey)。dkey 和 akey 都可以是可变长度的类型（字符串、整数或其它复杂的数据结构）。同一个 dkey 下的所有条目都保证被配置在同一个target上。 与 akey 关联的值可以是不能部分修改的单个可变长度值，也可以是固定长度值的数组。akeys 和 dkey 都支持枚举。

      Key-value API 提供了一个简单的键和可变长度值接口。它支持传统的 put、get、remove 和 list 操作。

      Array API 实现了一个由固定大小的元素组成的一维数组，该数组的寻址方式是 64 位偏移寻址。DAOS 数组支持任意范围的读、写和 punch 操作。

## DAOS 分布式异步对象存储｜控制平面
   DAOS 通过两个紧密集成的平面进行运转。数据平面处理繁重的运输操作，而控制平面负责进程编排和存储管理，简化数据平面的操作。

   DAOS 控制平面使用 Go 中编写，并作为 DAOS 服务器 (daos_server) 进程运行。除了实例化和管理在同一主机上运行的 DAOS 数据平面（引擎）进程外，它还负责网络和存储硬件的配置和分配。


## DAOS 分布式异步对象存储｜数据平面 
   DAOS 通过两个紧密集成的平面进行运转。数据平面处理繁重的运输操作，而控制平面负责进程编排和存储管理，简化数据平面的操作。
   ### 模块接口
      I/O 引擎支持一个模块接口，该接口允许按需加载服务器端代码。每个模块实际上都是一个库，由 I/O 引擎通过 dlopen 动态加载。模块和 I/O 引擎之间的接口在 dss_module 数据结构中定义。

      每个模块应指定：
         模块名
         daos_module_id 中的模块标识符
         特征位掩码
         一个模块初始化和销毁函数

      此外，模块还可以选择配置：
         在整个堆栈启动并运行后调用的配置和清理函数
         CART RPC 处理程序
         dRPC 处理程序

      I/O 引擎是一个多线程进程.
      I/O 引擎包括一个 dRPC 服务器，它监听给定 Unix Domain Socket 上的活动。
         dRPC 服务器定期轮询传入的客户端连接和请求。它可以通过 struct drpc_progress_context 对象同时处理多个客户端连接，该对象管理监听 Socket 的 struct drpc 对象以及任何活动的客户端连接。

         dRPC 进程, drpc_progress 表示 dRPC 服务器循环的一次流程。其工作流程如下：
            1.在监听 Socket 和任何打开的客户端连接上同时进行超时轮询。
            2.如果在客户端连接上看到任何活动：
               如果数据已输入：调用 drpc_recv 处理输入的数据。
               如果客户端已断开连接或连接被破坏：释放 struct drpc 对象并将其从 drpc_progress_context 中删除。
            3.如果在监听器上发现任何活动：
               如果有新的连接进入：调用 drpc_accept 并将新的 struct drpc 对象添加到 drpc_progress_context 中的客户端连接列表中。
               如果有错误：将 -DER_MISC 返回给调用者。I/O 引擎中会记录该错误，但不会中断 dRPC 服务器循环。在监听器上获取到错误是意外情况。
            4.如果没有看到任何活动，则将 -DER_TIMEDOUT 返回给调用者。这纯粹是为了调试目的，实际上，I/O 引擎会忽略此错误代码，因为缺少活动实际上并不是一种错误。




      

