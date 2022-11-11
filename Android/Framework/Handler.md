# Handler
- 消息管理机制：贯穿 Android 系统层。所有事务，点击、滑动、亮屏等

## ActivityThread

> 启动：launcher -> zygote -> Jvm -> ActivityThread.main

```
Looper.prepareMainLooper();
Looper.loop();

for 循环保证app运行
```


## 源码

- 流程

Handler -> post/send -> sendMessageAtTime() 发送消息

Looper -> loop() -> messageQuene.next() 获取消息

-> msg.target.dispatchMessage() 回调 Handler 的 dispatchMessage

- 消息如何保证顺序？
msg.when：Hander.sendMessage 时会给 when 赋值;

- 保证线程 Looper 的唯一性？

ThreadLocal 机制：

每一个线程存在 ThreadLocalMap 变量，用于储存线程上下文环境。
key 唯一（ThreadLocal 变量）,value 当前值也是唯一。
```
set(T value){
    Thread t = getCurrentThread();
    ThreadLocalMap map = getMap(t);
    if(map != null){
        map.setVal(this, value);
    } else {
        creatMap(t, value);
    }
}
```
MessageQueue 唯一性：
```
final MessageQueue queue = null; // final 修饰

Looper(){
    queue = new MessageQueue();
}
```

Handler 持有自己的 Looper 对象，也可以指定 Looper 来创建。变量在内存中，可以共享，而函数是执行在线程中。

## HandlerThread
- 特点：是一个线程，创建了 Looper 帮助处理同步

## IntentService
- 应用：多个任务，按照顺序执行。-> RxJava、IntentService 使工作在同一子线程，Queue 保证顺序，且优先级较高。

## 

## 问题
1. 独立 JVM 的原因？
   数据隔离、应用独立，某个程序挂掉不影响别的进程。

2. 一个线程有几个 Handler?
    可以创建多个 Handler，往 MessageQueue 中发送消息。MessageQueue 是在 Looper 创建的，Looper 在线程中是唯一的。

3. 一个线程有几个 Looper？
    ThreadLocal 

4. Handler 内存泄露？为什么别的比如 adapter 等不会？
    Handler 内存泄露 -> handleMessage 执行函数 -> 匿名内部类持有 Activity 的引用 -> msg.target 是 handler -> Activity 销毁，msg 还在内存中（延时消息）;
    adapter 等与 Activity 生命周期相同。
5. 子线程 Handler 没有消息时怎么处理？
    message.enque() 中返回 null，结束循环。调用 looper.quit 可以结束轮训。

6. Handler 有什么问题，优化方向？
没有做阻塞，可以无限放 Message，直到内存耗尽。
BlockingQueue -> 设置上限、无消息睡眠，有消息唤醒。
Handler 为什么不这样做：不止自己的消息到handler、系统的也会来，不设置上限。

7. Handler 等待机制？
    Message 队列为空，ptr = -1 nativePollOnOnce(-1) 无限等待。-> enqueneMessage() nativeWake 主动唤醒。
    下一条消息时间还没到，nativePollOnOnce 传入时间休眠一段时间。-> 到时间自动唤醒

synchronized ：内置锁 加锁和开锁的过程，都由 JVM 完成。

8. Message 的取出和回收
    Message.target.dispatch 之后，调用 recycyleUnchecked 将 Message 内容置空，放入 sPoll 链表头部；
    Message.obtain 获取头部结点的 Message；不用频繁创建对象、避免内存抖动；防止OOM；

9. 同步屏障
    ViewRootImp schelTravels 设置同步屏障 -> loop MessageQuene next if msg.target == null，进入 dowhile 循环 -> ViewRootImp 发送异步消息 -> dowhile 循环拿到异步消息 处理

    Chore ... VSync 垂直同步信号，post view 绘制消息到 Handler。特征是 target 为 null、isASync 为 true。
    MessageQueue next 的时候，do while 获取屏障类消息。为的是保证绘制事件插队处理，保证 View、用户交互优先给到用户。




