# fuse用户态文件系统框架

| 作者                 | 时间       | QQ技术交流群                      |
| -------------------- | ---------- | --------------------------------- |
| perrynzhou@gmail.com | 2020/05/25 | 中国开源存储技术交流群(672152841) |


## 基本介绍
- 文件系统提供了通用的应用程序的访问数据的接口，一般分为两种实现，一种是内核在用户态实现了文件系统；另外一种是内核在自己的内核态实现了文件系统，这也是内核的一部分，在内核态实现这个文件系统避免了消息在用户态和内核态之间的切换，具备比较高的性能
- 随着内核态文件系统(xfs/ext4)的复杂性的增加,在用户态开发文件系统变得比较流行，同时在用户态开发比较容易维护，即使crash了也不会导致kernel crash.如果是基于内核态文件系统xfs crash了，整个kernel就会crash.大部分用户态文件系统开发都是基于Fuse。

## Fuse架构

- FUSE是实现用户态文件系统的框架，其基本架构如下：
  ![fuse](../images/fuse_arch.PNG)
  Fuse有两部分组成：fuse驱动和用户态的daemon.fuse驱动是由内核的fuse设备驱动(/dev/fuse),这个字符设备驱动充当代理，针对不同的文件系统实现提供kernel和用户态daemon的通信桥梁；用户态daemon是从/dev/fuse设备读取，然后处理这些请求，最后把处理的就结果写回到/dev/fuse设备。

## Fuse工作流程
- 当应用程序在一个mount fuse的文件系统上执行操作，虚拟文件系统路由这个操作到fuse内核驱动，然后创建一个fuse request放到fuse的队列中，此时应用程序进程处于等待状态；fuse的用户态的daemon从/dev/fuse读取request，处理过程中damon需要陷入内核态读/dev/fuse设备，处理完成了把处理结果写入到/dev/fuse设备，最后在唤醒应用程序的进程

## Fuse请求类型

| 请求类别           | 请求类型                                                     |
| ------------------ | ------------------------------------------------------------ |
| Special            | init、destroy、interrupt                                     |
| Metadata           | loopup、forget、batch_forget、create、unlink、rename、open、release、statfs、fsync、flush、access |
| Data               | read、write                                                  |
| Attribute          | getattr、setattr                                             |
| Extended Attribute | setxattr、getxattr、listxattr、removexattr                   |
| Symlinks           | symlink、readlink                                            |
| Directory          | mkdir、rmdir、opendir、releasedir、readdir、readdirplus、fsyncdir |
| Locking            | getlk、setlk、setlkw                                         |
| Misc               | bmap、fallocate、mknod、ioctl、poll、notify_reply            |

- 当fuse的内核模块和daemon通信，会创建fuse request的结构。这写请求直接映射为vfs的操作
- init
  - 当mount一个fuse的文件系统，内核会产生一个init 请求，这时候用户空间和内核空间会确认协议的版本,readdirplus/flock等是否支持，还有一些fuse的参数(read-ahead大小)设置等。
- destroy
  - 当umount已经挂载的fuse文件系统，内核会创建这个请求，清理所有占用的资源。
- interrupt
  - 如果针对fuse文件系统发出一个请求，但是请求阻塞住了，在没有完成这个请求之前，终止这个请求，内核会产生interrupt请求。比如读取一个大文件，读取一半的时候，kill这个进程，interrupt请求就会被生成
- lookup
  - 每个请求在用户空间和内核空间都会包含一个64位整形的node id，l当文件路径转换为inode时候会产lookup请求，同时把inode缓存在内核的inode cache中
- forget
  - 当从内核inode cache(dcache)移除时候，内核给daemon传递一个forget的请求
- batch_forget
  - 允许内核在移除多个inode，并在一个请求中时候batch_forget请求会产生
- open
  - 当用户进程打开一个fuse文件系统中文件，open请求会产生
- flush
  - 当文件被close时候，flush请求会产生
- release
  - 当针对打开的文件不再被引用时，release请求会生成
- opendir和releasedir
  - 打开一个目录或者目录不再被引用,分别产生opendir和releasedir请求
- readdirplus
  - 当读取一个或者多个目录，并且获取每个目录元数据时候，readdirplus请求会产生
- access
  - 当用户访问一个文件，内核评估用户是否有权限访问这个文件，内核会产生access请求
  
### fuse 库

- fuse库有两部分组成，一个底层API 和上层API。
- 底层API
  - 从内核接受和解析请求
  - 发送结果
  - fuse文件系统配置和挂载
  - 隐藏内核和用户空间的差别
- 上层API
  - 在底层API之上跳过了文件到inode映射，直接操作的是文件或者路径
  - 使用文件操作的poxsix函数操作，比如chmod/chown等函数
  
### fuse队列

#### queue类型
![fuse_queue](../images/fuse_queue.JPG)

- fuse内核模块维护了5个请求队列,分别是interrupts/forgets/pending/processing/background.每一个请求仅仅会被放到一种类型的队列中
- interrupts：fuse把interrupt放到interrupt队列中
- forgets：forget请求放到forgets队列
- pending：同步请求(比如请求元数据)放到pending队列
- processing： daemon处理请求的队列
- background：异步请求放到background队列
- daemon从/dev/fuse读取处理流程
  - 在请求还没有到达daemon之前，每个请求设置一个优先级放到interrupts队列
  - forget和非forget请求被公平选择，每8个非forget请求和16个forget请求被传输，pending队列上最后面请求顺序移动processing队列，然后daemon进行处理，如果processing队列是空的,fuse daemon进程会被阻塞在read上，当fuse daemon回应响应，processing队列上的请求会被移除