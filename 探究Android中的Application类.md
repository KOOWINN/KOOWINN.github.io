一个Android App在运行时会自动创建全局唯一的`Application`对象，是用来维护应用全局状态的基类，它的生命周期等于应用的生命周期。

## 1. 主要方法
### (1) 构造函数
```Java
public class Application extends ContextWrapper implements ComponentCallbacks2 {

    public LoadedApk mLoadedApk;
    public Application() {
        super(null);
    }
}
```
`Application`拥有一个`LoadedApk`类型的成员变量，这个对象其实就是APK文件在内存中的表示。APK文件的相关信息，诸如其中的代码和资源，甚至Activity、Service等四大组件的信息都可以通过此对象获取。继续看构造函数，它直接调用了父类`ContextWrapper`的构造方法：
```Java
public class ContextWrapper extends Context {

    Context mBase;
    public ContextWrapper(Context base) {
        mBase = base;
    }
}
```
ContextWrapper是Context的包装类，mBase就是它的成员变量，除了通过构造函数初始化，也可以通过`attachBaseContext`方法初始化：
```Java
public class ContextWrapper extends Context {
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
}
```
上述方法被`Application`的`attach`方法用来实现实现了以下两个功能：①将新创建的ContextImpl对象赋给Application的父类成员变量mBase；②将新创建的LoadedApk对象赋给Application的成员变量mLoadedApk。
```Java
public class Application extends ContextWrapper implements ComponentCallbacks2 {
    final void attach(Context context) {
        attachBaseContext(context);
        mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
    }
}
```
这个方法因为是被系统调用的，所以是一个隐藏的final方法，我们无法重写。具体来说，它被系统的`Instrumentation`类所调用：
```Java
public class Instrumentation {
  public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    Application app = getFactory(context.getPackageName()).instantiateApplication(cl, className);
    app.attach(context);
    return app;
  }

  static public Application newApplication(Class<?> clazz, Context context)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    Application app = (Application)clazz.newInstance();
    app.attach(context);
    return app;
  }
}
```
`newApplication`方法被`LoadedApk`类所调用：
```Java
public final LoadedApk{

  public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    //保证了一个LoadedApk对象只创建一个对应的Application对象
    if (mApplication != null) {
        return mApplication;
    }

    ...

    Application app = null;

    ...

    try {
        //获取类加载器
        java.lang.ClassLoader cl = getClassLoader();
        if (!mPackageName.equals("android")) {
            initializeJavaContextClassLoader();
        }
        //创建ContextImpl对象
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        //创建Application对象
        app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) {
        ...
    }
    mActivityThread.mAllApplications.add(app);
    //将刚创建的Application对象赋值给mApplication变量
    mApplication = app;

    ...

    return app;
  }
}
```
可见，Application的父类方法`attachBaseContext(Context context)`中传入的实参其实是通过`ContextImpl.createAppContext(mActivityThread, this)`这句话所创建的ContextImpl对象。而且，加载和构造出Application对象的类加载器其实是由`LoadedApk.getClassLoader()`方法创建得到的，这里就不展开讲述了。紧接着，`ActivityThread`类调用了LoadedApk的`makeApplication`方法来初始化Application信息：
```Java
public final class ActivityThread extends ClientTransactionHandler {

    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            ···
        } else { //system进程才执行该流程

            android.ddm.DdmHandleAppName.setAppName("system_process",
                    UserHandle.myUserId());
            try {
                //创建Instrumentation
                mInstrumentation = new Instrumentation();
                mInstrumentation.basicInit(this);
                //创建ContextImpl
                ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
                //创建Application
                mInitialApplication = context.mPackageInfo.makeApplication(true, null);
                //回调Application的onCreate方法
                mInitialApplication.onCreate();
            } catch (Exception e) {
                throw new RuntimeException(
                        "Unable to instantiate Application():" + e.toString(), e);
            }
        }
    }

    private void handleBindApplication(AppBindData data) {
        ···
        final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);

        //创建Instrumentation对象
        if (ii != null) {
            ···
            final LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                appContext.getClassLoader(), false, true, false);
            ···
            final ContextImpl instrContext = ContextImpl.createAppContext(this, pi,appContext.getOpPackageName());
            ···
            final ClassLoader cl = instrContext.getClassLoader();
            mInstrumentation = (Instrumentation)
                cl.loadClass(data.instrumentationName.getClassName()).newInstance();
            ···
        } else {
            mInstrumentation = new Instrumentation();
            mInstrumentation.basicInit(this);
        }

        Application app;
        // 此处data.info是指LoadedApk, 通过反射创建目标应用Application对象
        app = data.info.makeApplication(data.restrictedBackupMode, null);
        ···
        mInitialApplication = app;
        ···
        mInstrumentation.callApplicationOnCreate(app);
        ···
    }
}

public class Instrumentation {
    public void callApplicationOnCreate(Application app) {
        //回调Application的onCreate
        app.onCreate();
    }
}
```
因为system_server进程和app进程都运行着一个或多个app，每个app都会有且仅有一个对应的`Application`对象(该对象和`LoadedApk`对象一一对应)。
system_server进程是由`ActivityThread.attach()`方法创建出Application对象的；
而普通app进程的则是由`ActivityThread.handleBindApplication()`方法创建得到。

### (2) onCreate()
从上述代码调用流程可以看到，Application初始化完成后系统就会回调其`onCreate`方法，该方法默认为空实现，我们可以扩展`Application`类，可以通过重写该方法再进行一些应用程序级别的资源初始化或临时全局共享数据的设置工作。

### (3) registerComponentCallbacks()和unregisterComponentCallbacks()
注册和注销`ComponentCallbacks2`监听器，下文将具体进行介绍。

### (4) registerActivityLifecycleCallbacks()和unregisterActivityLifecycleCallbacks()
注册和注销对所有`Activity`生命周期的监听器`ActivityLifecycleCallbacks`，每当应用程序内Activity的生命周期发生变化时，监听器中相对应的接口方法就会被调用执行。

### (5) onTerminate()
在应用程序结束时调用，但该方法只用于仿真机测试,真机上并不会调用。

## 2. `ComponentCallbacks2`监听器
Application已经实现了该监听器接口，如前文所述，可以通过registerComponentCallbacks()方法注册该监听器,也可以使用unregisterComponentCallbacks()注销。这个监听器提供了以下三个接口方法：
### (1) onTrimMemory(@TrimMemoryLevel int level)
- **作用**：指导应用程序根据当前系统内存使用情况释放自身资源（如图片或文件等缓存、动态生成和添加的View等），以避免被系统清除，提高用户体验。
>这是因为系统在内存不足时会从`LRU Cache`中按照从低到高的顺序杀死进程，但是那些高内存占用的应用也会被优先杀死，以此来让系统更快地获取更多可用内存。所以如果能够及时降低应用的内存占用，就可以降低它在后台被杀掉的概率，用户返回应用时就能快速恢复。

- **调用时刻**：当系统检测到当前进程适合进行无用内存的释放操作时。例如系统却已经没有足够的内存来维持目前所有的后台进程，而我们程序正好处于后台状态。

- **TrimMemoryLevel**：表明了系统在回调onTrimMemory方法时的内存情况等级：

    ①`TRIM_MEMORY_RUNNING_MODERATE`：级别为5，应用程序处于前台运行，但系统已经进入了低内存的状态。

    ② `TRIM_MEMORY_RUNNING_LOW`：级别为10，应用程序处于前台运行，虽然不会被杀死，但是由于系统当前可用内存很低，系统开始准备杀死其他后台程序，我们应该释放不必要的资源来提供系统性能，否则会影响用户体验。

    ③ `TRIM_MEMORY_RUNNING_CRITICAL`：级别为15，应用程序处于前台运行，大部分后台程序都已被杀死，此时我们应该尽可能地去释放任何不必要的资源。

    ④ `TRIM_MEMORY_UI_HIDDEN`：级别为20，应用程序的所有UI界面已经不可见，非常适合释放UI相关的资源。

    ⑤ `TRIM_MEMORY_BACKGROUND`：级别为40，应用程序处于后台，且在LRU缓存列表头部，不会被优先杀死，但是系统将开始根据LRU缓存来依次清理进程。此时应该释放掉一些比较容易恢复的资源提高系统的可用内存，让程序能够继续保留在缓存中。

    ⑥ `TRIM_MEMORY_MODERATE`：级别为60，应用程序处于后台，且在LRU缓存列表的中部，如果系统可用内存进一步减少，程序就会有被杀掉的风险。

    ⑦ `TRIM_MEMORY_COMPLETE`：级别为80，应用程序处于后台，且在LRU缓存列表的尾部，随时会被系统杀死，此时应该尽可能地把一切可以释放的资源释放掉。

- 除了Application，可以实现`onTrimMemory`回调的组件还有Activity、Fragement、Service、ContentProvider。

### (2) onLowMemory()
该接口在Android4.0以后被上述方法替代，所以作用和上述方法类似。
若应用想兼容Android4.0以前的系统就使用OnLowMemory，否则直接使用OnTrimMemory即可。需要注意的是，`onLowMemory`相当于level级别为`TRIM_MEMORY_COMPLETE`的`OnTrimMemory`。

### (3) onConfigurationChanged(@NonNull Configuration newConfig)
- **作用**：监听应用程序的一些配置信息的改变事件(比如屏幕旋转)
- **调用时刻**：当配置信息发生改变时。所谓的配置信息也就是我们在`AndroidManifest.xml`文件 中为Activity标签配置的`android:configChanges`属性值，例如`android:configChanges="keyboardHidden|orientation|screenSize`可以使该Activity在屏幕旋转时不重启，而是执行onConfigurationChanged方法。

## 3. 自定义Application类
- 新建自定义Application子类，继承Application类，选择重写相应的方法，如onCreate()方法；
- 在`AndroidManifest.xml`文件中配置`<application>`标签的`android:name`属性，例如`android:name=".MyApplication"`，MyApplicaiton就是自定义的Application类名。

## 4. 比较`getApplication()`和`getApplicationContext()`方法
- `getApplication()`方法只存在于`Activity`和`Service`对象中，该方法可主动获取当前所在mApplication，这是由`LoadedApk.makeApplication()`进行初始化的;
- `getApplicationContext()`是Context类的方法，所以Context的子类都可以调用该方法。先来看一下该方法的执行逻辑：
  ```Java
  public abstract class Context {
    public abstract Context getApplicationContext();
  }

  class ContextImpl extends Context {
      public Context getApplicationContext() {
          return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
      }
  }

  //上述mPackageInfo的数据类型为LoadedApk
  public final class LoadedApk {
      Application getApplication() {
          return mApplication;
      }
  }

  //上述mMainThread为ActivityThread
  public final class ActivityThread {
      public Application getApplication() {
          return mInitialApplication;
      }
  }
  ```
- 从上述代码可以看出如果`LoadedApk`非空，`getApplicationContext()`方法返回的就是`LoadedApk`的成员变量`mApplication`，所以对于Activity或Service组件来说， `getApplication()`和`getApplicationContext()`没有差别，因为它们的返回值完全相同。

- BroadcastReceiver和ContentProvider无法使用`getApplication()`，但是可以使用`getBaseContext().getApplicationContext()`获取所在的Application，但是ContentProvider使用该方法有可能会出现空指针的问题, 情况如下:
当同一个进程有多个apk的情况下, 如果第二个apk是由provider方式拉起，那么 由于provider创建过程并不会初始化相应的Application，此时执行`getContext().getApplicationContext()`就会返回空，所以对于这种情况需要做好判空处理。

- 在Context对象的`attachBaseContext()`中调用`getApplicationContext()`方法也会返回空，这是因为从前文对Application构造函数的分析可知，`LoadedApk.mApplication`是在`attachBaseContext()`方法执行之后才被赋值的。

参考文章：
1. [理解Application创建过程](http://gityuan.com/2017/04/02/android-application/)
2. [一起来学习那个熟悉又陌生的Application类吧](https://mp.weixin.qq.com/s/jIfJTx1HZvB0q7AFsZ42bQ)