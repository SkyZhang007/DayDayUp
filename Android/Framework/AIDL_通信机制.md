# AIDL 是什么
- DSL：特殊领域语言；
- 配置文件：跨进程通信的配置；
- 编译器、翻译器：可翻译为 Java 文件使用。

![AIDL 流程](https://upload-images.jianshu.io/upload_images/6762021-c10f7f05e0289846.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## asInterface
asInterface 会执行 queryLocalInterface，如果能够查到则返回本地Binder，反之创建代理。


# 四大组件通信流程（大概）

经过六次跨进程通信

1. 客户端 -> ServiceManager ==> AMS 服务的 IBinder（注意此时只是拿到了AMS 的 Binder 代理）;
2. 客户端通过 AMS IBinder -> AMS 进行通信 ==> AMS 在 SM 进程，请求 bindService；
3. AMS -> 服务端进行通信 ==> 执行 Service onBind，此时拿到了服务端的 IBinder；
4. 服务端 -> AMS（SM进程） ==> 为的是把自己的 IBinder 交给 AMS；
5. 服务端把自己的 IBinder 交给 AMS ==> 涉及到与 AMS 通信；
6. AMS 拿到服务端的 IBinder，回调给 客户端(onServiceConnected)。





