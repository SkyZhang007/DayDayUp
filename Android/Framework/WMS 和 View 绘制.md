

# WMS 

## 前言
- ServiceManager 是一个独立进程,守护进程，Binder 的管理者，所有进程把自己的 Binder publish 到这里；
- AMS、WMS 都是 Service，由 SystemServer 进程启动，SystemServiceManage 管理；这些服务的获取，是通过获取 ServiceManager 的 Binder 代理来获取，是一个 IPC 过程。
- 进程创建过程 

![创建流程](https://upload-images.jianshu.io/upload_images/6762021-809cc6de0eaa65ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 应用要启动，首先拿到 ServiceManager 的 Binder 代理。然后在 ServiceManager 中查询注册过的服务的 Binder，比如这里去查询 AMS 的 Binder（SystemServer 启动时将这些服务的 Binder 注册publish到 ServiceManager进程。）
2. AMS 发送 Socket 请求到 Zygote 进程进行 fork，之后通过反射执行 ActivityThread 的 main 函数进行 attach；（main 函数 Looper.loop 进行死循环 保证app一直运行）
3. attach 会将 app 进程的 Binder 代理 ApplicationThread（），attachApplication 提供给 AMS 共享，相当于注册到 AMS，跨进程通信时再通过 AMS 获取。(此时进程将 pid 等参数给 AMS 管理，安全性)

## View 相关

![View 层级和刷新](https://upload-images.jianshu.io/upload_images/6762021-aa65de98fe4f2604.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- invalidate 请求重绘
![重绘流程](https://upload-images.jianshu.io/upload_images/6762021-16adfd5775b7010b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![View 绘制流程](https://upload-images.jianshu.io/upload_images/6762021-52e3213af47b47a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. invalidate -> invalidateInteral -> 设置标志位 Flag -> 找到父 View，执行 p.invalidateChild；
2. ViewGroup -> 设置标志位 Flag，设置可能要刷新的区域（与子 View 重合的部分 dirty 区域）；
3. ViewGroup 再找到自己的 parentView，执行重绘 parent.invalidateChildInParent，while 循环操作；
4. 最后调用根布局 ViewRootImp 的 invalidateChildInParent，计算好最终要刷新的范围 -> invalidateRectOnScreen() -> scheduleTraversals() 。*注释2

## WMS 流程
1. ActivityThread 启动，调用 handleLaunchActivity -> performActivity(反射创建Activity) -> activity.attach() 创建 Window（Activity、Dialog 等要显示，必然需要Window）*注释3
2. 为创建的 PhoneWindow 设置 WindowManager。（WMS，由 WMS 管理 Window）
3. callActivityOnCreate() -> setContentView() 解析 xml、创建 DecorView 并将解析的 View 添加（实现动态换肤，可以在此处定义 Factory 加载自定义资源）。
4. 接着 performOnResume() -> onResume() 之后，当前 Activity 的 PhoneWindow 才会添加 DecorView -> 底层绘制View




## 问题
- invalidate 是否触发其它 View 重绘？



> * 注释1 invalidateInteral # skipInvalidate()
> skipInvalidate 会进行判断，如果某 Activity 在后台，这里会return 跳过绘制。
> * 注释2
> DecorView 是应用最底层View 负责管理子 View/ViewGroup，ViewRootImp 负责刷新 View。
> scheduleTraversals 会将绘制 Runnable post 到 Handler，等到下次刷新时（16ms）执行 performTraversals：measure layout draw 流程。(如何保证下次首先刷新view？Barrier 同步屏障机制)
> * 注释3
> Window 类型：大致分为三种 -》 app Window、子窗口、系统 Window