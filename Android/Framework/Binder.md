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