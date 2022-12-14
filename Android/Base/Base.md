- Bundle
    - 内部结构，ArrayMap key 和 value 都使用数组，查找使用二分法


# startActivity
1. 主要是应用进程和 system_server 的 AMS 进行通信的过程，AMS 管理着 Activity 的启动、Activity栈、启动模式等等。startActivity 的大概流程就是应用进程 IPC 调用到 AMS，AMS 处理完再 IPC 回到应用进程，之后创建 Activity 实例，回调生命周期。

2. 跨进程的实现：AIDL，应用程序进程到 AMS 所在进程通过 IAMS.aidl、回来则通过 IApplicationThread.aidl。

3. startActivity 生命周期：onPause -> onCreate start resume -> onStop.
怎么保证执行顺序：
AMS 控制，启动事务通过 IPC 发送到 app 进程的 ActivityThread.H，是一个 Handler，事务在队列 MessageQueue 中按顺序执行。

4. 启动插件 Activity：manifest 注册问题，占位然后替换


# UI线程
1. UI线程一定是主线程吗？
   不一定，对于 Activity 来说是的，runOnUiThread 就是判断当前线程是不是成员变量 mUiThread。这个变量在 Actiivty 创建的 attach 时赋值。
   对于View来说， ui 线程是 ViewRootImp 创建时的线程。view.post() 执行 attachInfo.mHander.post()，attachInfo 是ViewRootImp 创建时赋值。
2. 为什么 post 可以获取 View 宽高？
   ViewRootImp 在 onResume 创建，可能出现获取不到的情况。这时候把 action 放到 Queue(像排队，队尾添加 队头取的顺序) 中，ViewRootImp 执行 performTravels 执行这些 Action。此时已经完成的 measure layout draw 流程。
3. 