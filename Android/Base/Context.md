
# Context
- **通俗解释**
  上帝对象、获取资源(LoadedApk)、启动组件(ActivityThread)；
  Context 抽象类、ContextImp 实现功能、包装 Wapper 保护扩展。Activity Service Application 继承之，Activity 继承带 Theme 的Wapper。

- **getApplication() getApplicationContext()**
  getApplication Activity、Service 创建时 attach 的成员变量，getApplicationContext ContextImp 获取；
  Activity 创建：ClassLoader 加载、反射创建、LoadedApk makeApplication、createBaseContextForActivity 给 Activity 创建 contextImp、attach 变量赋值 application。
  ContextImp 获取 Application 也是从 LoadedApk 获取，其实是同一个。

- **ContentProvider 的 Context**
  ContentProvider 创建时通过构造函数赋值变量。Application 创建的时候调用 installContentProvider 创建、之后回调 Application 的 onCreate。

- **BroadcastReceiver 的 Context**
  动态注册的时候持有 Context，回调 onReceive 的时候带入 ==> 所以需要取消注册。
  静态注册分发时调用 ActivityThread.handleReceiver，由 ClassLoader 创建 Reveiver 



1. 一句话说明："上帝对象"，获取资源、调用四大组件等。
2. 模拟类图
抽象：                   Context
实现： ContextWapper(ContextImp 的包装)       ContextImp(核心逻辑)
子类：Application Service  ContextThemeWapper
                            Activity
四大组件持有 mBase 对象，获取的就是 Wapper 保存的 ContextImp 对象，进而调用 ContextImp

1. ContextImp 核心参数：ActivityThread mMainThread
```
    @Override
    public Looper getMainLooper() {
        return mMainThread.getLooper();
    }

    @Override
    public Context getApplicationContext() {
        return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
    }
    
    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();

        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }
    
    //差不多有10个类似方法
    @Override
    public void sendBroadcast(Intent intent, String receiverPermission, int appOp) {

            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, receiverPermissions, appOp, null, false, false,
                    getUserId());

    }
```
    ContextImp 核心参数：LoadedApk mPackageInfo
LoadedApk 可以看作 apk 文件在内存中的表现，通过该对象获取apk信息、资源。
1. 