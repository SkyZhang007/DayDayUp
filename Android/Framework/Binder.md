> [Binder 总览](http://gityuan.com/2015/10/31/binder-prepare/)
> [谈谈你对 Binder 的理解](https://zhuanlan.zhihu.com/p/152237289)
> [Binder](https://mp.weixin.qq.com/s?__biz=MzIzOTkwMDY5Nw==&mid=2247483707&idx=1&sn=e797e0d67bc4e3b8ef78ef5a64adbac9&chksm=e922404dde55c95b357f3da9ca3473534390ad7feae7acb5d07615f2237ecdd048f73de6c205&scene=178&cur_album_id=1343624103303102464#rd)
# Binder是什么
- 进程间通信；
- 虚拟的物理驱动(没有硬件)；
- Java层跨进程调用；

# 为什么使用 Binder
- 性能：mmap 单次拷贝；
- 安全性：进程 pid 等信息、CS 架构；

# Binder Native 层的启动
ProcessState.open_driver()：ProcessState 是单例模式，保证进程一个Binder

1. init 创建 "dev/binder" 节点；
2. open 获取 Binder 描述符fd（一切皆文件的描述符）；
   ioctr 使用命令沟通 binder 驱动，设置 Binder 线程容量
3. mmap 内存映射；
   首先在内核创建一块与用户空间大小一致的虚拟内存，然后再申请1个page大小的物理内存(4kb 使用时扩容)，之后分别映射到用户和内核虚拟内存。实现内核空间的 Buffer 和用户空间 Buffer 同步操作的功能。
4. ioctr 传递参数给 Binder 驱动。
   ioctr(描述符，ioctr命令，数据类型) 作为驱动需要命令驱使，读写数据的过程需要持有同步锁；
   **binder_ioctl_write_read()**：也就是 copy_from_user和copy_to_user 的过程，缓存中存在数据则进行相应的读写操作。

> 四个方法都是通过用户层调用 syscall 执行到内核层的方法。
> system_server 进程 open、mmap 之后 ioctr 设置为 Binder 管家，之后 loop 等待操作；
> 普通进程 open、mmap 之后添加服务到 system_server。

# Binder 传输数据过程

![整体通信过程](https://upload-images.jianshu.io/upload_images/6762021-482408d7233832fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **BC_XXX**: BINDER_COMMAND_PROTOCOL，IPC 发往 Binder 用于执行命令，生成 binder_work；
- **BR_XXX**：Binder 响应码，用于 Binder 到 IPC 层。
  
Binder 通信至少是两个进程的交互：
- Client 进程执行 binder_thread_write 写入命令和数据，生成 binder_work:
- Server 进程执行 binder_thread_read  生成 BR_XXX 发送到用户空间处理。

![通信](https://upload-images.jianshu.io/upload_images/6762021-1b828ea63d47eb8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 注册服务过程 其实也是一次 IPC 过程
普通进程启动之后，通过 open/mmap 等之后，获取 defaultServiceManger.addService （BpServiceManager），如果此时 ServiceManager 没有创建，会执行创建。

注册的什么东西？ 服务名+对象（例如 "media.player" 和 MediaPlayerService）
注册的过程是由 Parcel 打包的数据，通过 write 写入。
![IPC过程](https://upload-images.jianshu.io/upload_images/6762021-70f6d44b4d62a17a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注册服务过程
1. MeadiaService 进程调用 ioctr 向 Binder 驱动发消息，消息里是 Parcel 打包的 handle（目标进程）、服务名、MeadiaService 对象；
2. Binder 驱动收到消息后，生成 BR_TRANSACTION 命令，找到执行该命令的线程（handle == 0 ServiceManager），把事项加入到该线程的 todo 队列；
3. ServiceManager 处理完注册以后，生成 BC_REPLY 应答命令给 Binder 驱动
4. Binder 收到以后生成 BR_REPLY 到 MeadiaService 进程，说明可以正常工作。


# Binder 线程池

Binder 设计中，只有第一个Binder主线程(也就是Binder_1线程)是主动创建的，Binder 线程池的普通线程都是根据需要创建并加入线程池。
Binder 线程池默认数量 15；
当发生 IPC 时，执行 binder_thread_read 时发现没有可用线程且线程数量没有达到上限，则创建线程。



# ServiceManager 的启动

要使用系统 Binder，需要从 ServiceManager 获取。在这之前，ServiceManager 需要启动。
> service_manager.c
```
int main{
    bs = binder_open(128*1024); // ->open("dev_binder")
    bs->mapped = mmap(...); // 驱动和sm的区域做映射
    binder_become_context_manager(bs); // 设置SM为管家->ioctl(BINDER_SET_CONTEXT_MGR)
    binder_loop(); // 循环监听
}
```

1. **binder_open**：打开驱动（默认分配内存 128K），内存映射；
2. **binder_become_context_manager**：设置 SM 为 Binder 管家；
```
int binder_become_context_manager(struct binder_state *bs)
{
    //通过ioctl，传递BINDER_SET_CONTEXT_MGR指令 --> 该指令最终调用 binder_ioctl_set_ctx_mgr
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```
3. **bind_loop** 轮询处理，准备完成，等待Client事件；**数据传输的过程主要还是 copy_from_user，write_size > 0 执行 binder_write、read_size > 0 执行 binder_read。**
    1. 执行 BC_ENTER_LOOP 命令，调用 **binder_ioctr_write_read；** *注释1
    2. 当 write_size > 0 -> binder_thread_write、read 同理；
    3. binder_thread_write、read 会 switch 命令进行处理，BC_ENTER_LOOP 命令设置 looper 状态为循环；

# ServiceManager 的获取

当需要从 ServiceManager 添加和获取服务时，需要获取 gDefaultServiceManager(Native)，是个单例。
```
sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
    {
        AutoMutex _l(gDefaultServiceManagerLock); //加锁
        while (gDefaultServiceManager == NULL) {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));
            if (gDefaultServiceManager == NULL)
                sleep(1);
        }
    }
    return gDefaultServiceManager;
}
```
1. **ProcessState::self():** 创建 ProcessState，重要参数和函数：
   open_driver：启动 Binder 驱动，返回 binder fd（文件操作符）
   **BINDER_VM_SIZE** 1M-8k 页缓存*2，设置Binder数据容量；
   mDriverFD：记录 binder 的 fd，用于调用驱动。
2. **getContextObject(NULL):**获取 BpBinder 对象：new BpBinder()；
3. IServiceManager::asInterface == new BpServiceManager(new BpBinder)
   BpServiceManager，











![Binder启动](https://upload-images.jianshu.io/upload_images/6762021-e1dd501eb6e33d13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- service_manager 的 Binder：进程创建之后，通过 binder_open 打开 dev/binder、
    通过 mmap 进行内存映射、注册为 BInder 管理者、循环等待消息
- system_server：Zygote fork 的进程，执行自己的 ZygoteInit() 方法 -> nativeZygoteInit -> ProcessState.self() 打开Binder 驱动，创建内存映射。

- 针对用户进程：与 AMS 通信创建 Process，执行 Process.start() 向 Zygote 进程发消息 fork 进程、新进程调用 nativeZygoteInit 到达 native 层执行 app_main.cpp 的 onZygoteInit()。接下来调用 ProcessState::self() 打开 binder 驱动（dev/binder）






linux 一切皆文件；
![Binder Open](https://upload-images.jianshu.io/upload_images/6762021-3810dec03ec56232.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. binder_init：
    1) 分配内存、初始化、把 binder 设备添加到链表（binder_services）
    2) **binder_fops:**注册调用驱动层的函数，通过 SystemCall

2. binder_open：  
    1) 创建 binder_proc 对象，保存当前进程信息；
    2) 添加到 binder_procs 链表中；

3. binder_mmap:
    1) 根据用户空间虚拟内存大小，分配一块内核虚拟内存（内存一样大）
    2) 分配一块 4kb 一页的物理内存（到使用时动态扩容）
    3) 把这块物理内存，分别映射到用户空间和内核空间的虚拟内存

4. binder_ioctl：
    1) 读写操作 cmd 命令 - BINDER_WRITE_READ


# Binder 服务获取

![service_manager](https://upload-images.jianshu.io/upload_images/6762021-f28b89d78d8efe60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## ServiceManager

### ServiceManager初始化
- 启动：Init 进程解析 init.rc 脚本，启动 ServiceManager 进程

> service_manager.c
```
int main{
    bs = binder_open(128*1024); // ->open("dev_binder")
    bs->mapped = mmap(...); // 驱动和sm的区域做映射
    binder_become_context_manager(bs); // 设置SM为管家->ioctl(BINDER_SET_CONTEXT_MGR)
    binder_loop(); // 循环监听
}
```

1. 打开驱动（默认分配内存 128K），内存映射；
2. 设置 SM 为 Binder 管家；
    1. 创建 binder_node 结构体对象；
    2. proc -> binder_node (BBinder);
    3. 创建 work todo 队列=>类似 MessageQueue
3. bind_loop 轮询处理，准备完成，等待处理事件；
    1. 执行 BC_ENTER_LOOP 命令，调用 **binder_ioctr_write_read；** *注释1
    2. 当 write_size > 0 -> binder_thread_write、read 同理；
    3. binder_thread_write、read 会 switch 命令进行处理，BC_ENTER_LOOP 命令设置 looper 状态为循环；

   
```
struct binder_write_read {
    binder_size_t     wirte_size;// byte to write
    binder_uintptr_t  write_buffer;// 数据
    binder_size_t     read_size;// byte to write
    binder_uintptr_t  read_buffer;// 数据
}
```

### ServiceManager Native 的获取

需要 SM 的情况：通过它注册服务或者获取服务：
> IServiceManager#defaultServiceManager() 方法，gDefaultService 是单例

1. ProcessState.self()  ---> getContextObject()
   1) 打开驱动 open_driver()：Binder；
   2) 设置线程最大数目 15；
   3) mmap 内存映射(普通服务大小 1M-8k)
    getContextObject
   4) 创建 BpBinder：native 层 SM 的对象为 BBinder、BpBinder 是它的代理对象，由调用者获取;
2. interface_cast
    1) BpService(new BpBinder)
    2) remote.transact --> remote == BpBinder
3. Java 层
    1. new ServiceManager(new BinderProxy)
    2. mRemote == BinderProxy
    3. BinderProxy.mObject == BpBinder
    4. mRemote.transact == BpBinder.transact





![调用流程](https://upload-images.jianshu.io/upload_images/6762021-ed2186844656b16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)






> 注释1 binder_ioctr_write_read 很重要，往 binder 驱动读写数据，都依靠此方法





















# AMS 进程创建流程
1. ATMS startProcess -> AMS ProcessList 查询进程-> 不存在调用 Process.start -> ZYGOTE.start、startViaZygote
-> zygoteSendArgsAndGetResult、**zygoteWriter.write(msgStr) zygoteWriter.flush()** 发送 Socket -> Zygote Socket 收到消息 fork 进程。
2. Fork 完成之后，创建 ZygoteInit 返回 Runnable 并 run、这个线程会初始化 Runtime，调用 nativeZygoteInit 启动 Binder;
3. (AndroidRuntime.cpp) nativeZygoteInit onZygoteInit -> (App_main.cpp)AndroidRuntime.onZygoteInit -> ProcessState.self -> openDriver 初始化 Binder -> proc.startThreadPoll （初始化 Binder 线程池）


# SM 管理系统进程 Binder
Android 10 之前，四大组件全部由 AMS 管理，之后由 ATMS 管理；
1. 管理App进程：ProcessList，进程的容器。
ProcessList 中又有 List<ProcessRecord> 储存正在运行的进程；*注释1
2. 启动 Activity 为什么要到 AMS？ Activity运行在进程中，首先要到 AMS 获取当前进程，才能启动；
3. 系统进程启动过程，有的调用 publishBinderService -> ServiceManager.addService()，而ATMS 直接 addService()：接下来获取到 ServiceManager 的 Binder 代理进行跨进程通信，将这些服务的 Binder 注册到 ServiceManager；

# AMS 管理 app 进程 Binder

![AMS 管理 app 进程](https://upload-images.jianshu.io/upload_images/6762021-2d0f04d9547fc603.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. ActivityThread main 函数会执行 attach 函数，将 ApplicationThread （Binder 代理）注册到 AMS


> 注释1：ProcessRecord 是进程的代表；ActivityRecord 是 Activity 在底层的表示；
> ActivityThread 代表一个进程，在底层表示为 ProcessRecord；

# Intent 传递数据最大 1m-8k(页缓存X2)？
Binder 定义、大文件 mmap 可解





