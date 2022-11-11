# Binder是什么
- 进程间通信；
- 虚拟的物理驱动(没有硬件)；
- Java层跨进程调用；

# 为什么使用 Binder
- 性能：mmap 单次拷贝；
- 安全性：进程 pid 等信息、CS 架构；

# Binder 的启动

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



### SM 服务添加过程

> ServiceManager.java -> getIServiceManager().addService(name, service, flase)

- getIServiceManager --- new ServiceManagerProxy(new BinderProxy())
    - ServiceManagerNative.asInterface(BinderInternal.getContextObject())
        - BinderInternal.getContextObject -- 返回 BinderProxy 对象
          - ProcessState::self() -> getContextObject: 创建 BpBinder
          - javaObjectForIBinder -- BinderProxy和BpBinder 互相绑定
        - ServiceManagerNatice.asInterface
          - 返回 ServiceManagerProxy
- addService
  - data.writeStrongBinder(service); -- service==AMS  将 AMS 放入 data 中
    



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





