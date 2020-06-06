## Java虚拟机与Android虚拟机
#### Java虚拟机
为了实现程序的“一次编写，到处运行”，Java技术体系提出了Java虚拟机（JVM）的概念。“虚拟机”是一个相对于“物理机”的概念，这两种机器都有代码执行能力，区别在于物理机的执行引擎是直接建立在处理器、缓存、指令集和操作系统层面上的，而虚拟机的执行引擎则是由软件自行实现，因此可以不受物理条件制约地定制指令集与执行引擎的结构体系，能够执行那些不被硬件直接支持的指令集格式。
此外，JVM不与包括Java语言在内的任何程序语言绑定，它只与“Class文件”这种特定的二进制文件格式所关联，Class文件中包含了Java虚拟机指令集、符号表以及若干其他辅助信息。因此，JVM兼具平台无关性和语言无关性。

#### Dalvik虚拟机
Dalvik虚拟机是一款专门为内存和处理器速度有限的移动设备所设计的Android虚拟机。Dalvik虚拟机并不是一个JVM，因为它没有遵循Java虚拟机规范，例如不支持直接执行Class文件、使用寄存器架构而不是栈架构等，但是它又与Java有很多关联，例如它支持执行的DEX（Dalvik Executable）文件是基于Class文件转化而来、应用程序可使用Java语法编写、可以直接使用绝大部分的Java API等。
>- 为什么采用Dex格式？这是专为Dalvik设计的一种压缩格式，Android使用dx工具将所有的class文件进行翻译、重构、解释、压缩等一系列处理后打包为一个或多个dex文件，这样可以降低安装包的体积。
>- 为什么使用寄存器架构？基于栈意味着需要频繁地从栈上读写数据，增加了指令分派和内存访问次数；而基于寄存器的方案使得数据的访问可以直接通过寄存器传递，极大提升了数据的读取速度。

#### ART虚拟机
ART即Android运行时环境（Android Runtime），一开始在Android 4.4系统中作为测试功能对外发布，并从Android 5.0开始全面取代Dalvik成为正式的Android虚拟机。相比于Dalvik，ART拥有更高的性能。下图给出了Dalvik和ART虚拟机工作流程上的差异：
![Dalvik&sART.png](https://user-gold-cdn.xitu.io/2020/6/1/1726fbaaca91f1c4?w=2810&h=2560&f=png&s=141005)

#### Dalvik与ART的区别
Dalvik虚拟机采用JIT(Just-In-Time)即时编译策略，这属于动态编译，APP在安装时启动`dexopt`过程对dex文件进行`verification`和`optimization`操作得到优化后的odex文件；APP在运行时每当遇到一个新类JIT编译器都会对它进行即时编译，经过编译后的代码会被优化成相当精简的原生型指令码（即native code），从而提高了下次相同逻辑的执行速度。但是，JIT存在以下缺陷：①每次启动应用都需要重新编译②运行时比较耗电，会造成电池额外的开销。

ART虚拟机则采用AOT(Ahead-Of-Time)提前编译策略，这属于静态编译，APP在安装时会启动`dex2oat`过程把dex预编译成本地可执行的ELF文件，每次运行程序的时候不用重新编译。但是AOT有以下两点缺陷：①应用安装和系统升级之后的应用优化比较耗时②优化后的文件会占用额外的存储空间

所以Android7.0开始ART虚拟机采用了AOT/JIT混合编译策略，应用在安装时dex不会被编译，在运行时dex文件先通过解析器被直接执行（这一步骤跟 Android 2.2 - Android 4.4之前的行为一致），与此同时，热点函数（Hot Code）会被识别并被JIT编译后存储在`jit code cache`中并生成 profile 文件以记录热点函数的信息；当手机进入IDLE（空闲）或者Charging（充电）状态时，系统会扫描App目录下的profile文件并执行AOT过程进行编译。

## Android类加载
### Java虚拟机类加载机制
JVM把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这个过程被称作虚拟机的类加载机制。类加载包含以下几个过程：
![JVM类加载流程.png](https://user-gold-cdn.xitu.io/2020/6/1/1726fbc00249abe3?w=764&h=271&f=png&s=14955)
负责完成上述工作的就是类加载器，即ClassLoader。该类有三个重要的方法 `loadClass()`、`findClass()` 和 `defineClass()`。
`loadClass()`方法是加载目标类的入口，所以先来看一下这个方法：
```Java
public abstract class ClassLoader {
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
    {
        // 首先判断请求的类是否已经被加载
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    // 若父加载器为空则默认使用启动类加载器作为父加载器
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // 如果父类加载器抛出ClassNotFoundException，则说明父类加载器无法完成加载请求
            }

            if (c == null) {
                // 在父类加载器无法加载时再调用自身的findClass方法进行类加载
                c = findClass(name);
            }
        }
        return c;
    }
}
```
这段代码的逻辑非常清晰：先检查请求加载的类型是否已经被加载过，若没有则调用父加载器的`loadClass()`方法，若父加载器为空则默认使用启动类加载器作为父加载器。假如父类加载器加载失败，抛出ClassNotFoundException异常的话，才调用自己的findClass()方法尝试进行加载。这其实就是类加载器的双亲委派机制，具体就是：
>如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求**委派给父类加载器**去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去完成加载。这样设计的好处是Java中的类随着它的类加载器一起具备了一种带有优先级的层次关系。

因此，我们在自定义类加载器时应该避免重写`loadClass()`方法，而应该重写`findClass()`方法，然后再调用`defineClass()`方法将字节码转换成Class对象。

JVM内置了3个重要的类加载器，绝大多数Java程序都会使用到它们来进行加载。
- **启动类加载器（BootstrapClassLoader）**：负责加载JVM核心类，这些类位于`<JAVA_HOME>\lib`目录下能被虚拟机识别的类库中，Java程序无法直接引用该类加载器。
- **扩展类加载器（ExtensionClassLoader）**：负责加载JVM扩展类，这些类位于`<JAVA_HOME>\lib\ext`目录内，开发者可直接使用扩展类加载器。
- **应用程序类加载器（ApplicationClassLoader）**：负责加载用户类路径（ClassPath）上所有的类库，我们自己编写的代码以及使用的第三方 jar 包通常都是由它来加载的。开发者可通过`ClassLoader.getSystemClassLoader()`方法直接获取，故又称为系统类加载器。当应用程序没有自定义类加载器时，默认采用该类加载器。

### Android中的类加载机制
Android系统实现了一个ClassLoader的子类——`BaseDexClassLoader`, 这个类又继续派生出了两个子类——`PathClassLoader`和`DexClassLoader`。

#### 类加载器
1.PathClassLoader

官方介绍：
>Provides a simple {@link ClassLoader} implementation that operates on a list of files and directories in the local file system, but does not attempt to load classes from the network. Android uses this class for its system class loader and for its application class loader(s).

PathClassLoader可加载本地文件系统上的类，被Android用来作为其系统类和应用类的加载器。

```java
public class PathClassLoader extends BaseDexClassLoader {

    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }

    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }

    ···
}
```

2.DexClassLoader

官方介绍：
>A class loader that loads classes from {@code .jar} and {@code .apk} files containing a {@code classes.dex} entry. This can be used to execute code not installed as part of an application.
Prior to API level 26, this class loader requires an application-private, writable directory to cache optimized classes.

DexClassLoader可以加载jar包和apk中dex文件包含的类，其实还包括zip文件或者直接加载dex文件，它可以被用来执行未安装的代码或者未被应用加载过的代码。

```java
public class DexClassLoader extends BaseDexClassLoader {

    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}
```
从上述两个类的构造函数可以看出，它们都调用了父类的构造函数：
```java
public class BaseDexClassLoader extends ClassLoader {

    private final DexPathList pathList;
    protected final ClassLoader[] sharedLibraryLoaders;

    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        this(dexPath, librarySearchPath, parent, null, false);
    }

    ···

    public BaseDexClassLoader(String dexPath,
            String librarySearchPath, ClassLoader parent, ClassLoader[] sharedLibraryLoaders,
            boolean isTrusted) {
        super(parent);
        // Setup shared libraries before creating the path list. ART relies on the class loader
        // hierarchy being finalized before loading dex files.
        this.sharedLibraryLoaders = sharedLibraryLoaders == null
                ? null
                : Arrays.copyOf(sharedLibraryLoaders, sharedLibraryLoaders.length);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);

        if (reporter != null) {
            reportClassLoaderChain();
        }
    }
}
```
构造函数中各参数的意义如下：
- dexPath: 需要加载的文件列表，文件可以是包含了classes.dex的 jar/apk/zip，也可以直接使用 classes.dex 文件，多个文件用 “:” 分割
- optimizedDirectory: dex文件优化的产物存在的目录
- librarySearchPath: native库所在路径列表;当有多个路径则采用:分割
- parent:父类的类加载器

DexClassLoader可以指定`optimizedDirectory`参数，所以dex文件优化后的产物会被存储在用户指定的这个目录中；而PathClassLoader无法指定，所以只能使用默认的目录`/data/dalvik-cache/`。需要注意的是，应用在安装时其内部的dex文件会被自动提取出来并进行优化，且优化后的产物被存储在上述默认目录中。从上面的分析可以得出，PathClassLoader其实并不是只能加载安装到系统上的apk，也可以加载其他 dex/jar/apk 文件，只不过dex文件优化后的产物只能存储在系统默认路径下。

不过，从Android8.0开始DexClassLoader中的`optimizedDirectory`参数也成了无效参数，这就意味着`PathClassLoader`和`DexClassLoader`使用上的差异性被消除了。

BaseDexClassLoader持有一个DexPathList对象，并在构造函数中进行了初始化。DexPathList非常重要，它内部维护了两个数组：dex文件数组和native库数组。dex文件数组基于`PathClassLoader`或`DexClassLoader`传入的`dexPath`参数生成，native库数组则基于`librarySearchPath`参数生成。

#### 类加载过程
由于PathClassLoader和DexClassLoader只是对BaseDexClassLoader的简单继承，所以接下来直接通过分析BaseDexClassLoader来梳理Android中类加载逻辑。作为ClassLoader的子类，BaseDexClassLoader重写了findClass()方法：
```java
// BaseDexClassLoader
public class BaseDexClassLoader extends ClassLoader {
  protected Class<?> findClass(String name) throws ClassNotFoundException {

          if (sharedLibraryLoaders != null) {
              for (ClassLoader loader : sharedLibraryLoaders) {
                  try {
                      return loader.loadClass(name);
                  } catch (ClassNotFoundException ignored) {
                  }
              }
          }
          ···
          Class c = pathList.findClass(name, suppressedExceptions);
          ···
          return c;
      }
}

// DexPathList
public final class DexPathList {

  private Element[] dexElements; //记录所有的dexFile文件
  NativeLibraryElement[] nativeLibraryPathElements; //记录所有的Native动态库文件

  public Class<?> findClass(String name, List<Throwable> suppressed) {
      for (Element element : dexElements) {
          Class<?> clazz = element.findClass(name, definingContext, suppressed);
          if (clazz != null) {
              return clazz;
          }
      }

      if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
      }
      return null;
  }
  ···
  // Element
  static class Element {
      ···
      public Class<?> findClass(String name, ClassLoader definingContext, List<Throwable> suppressed) {
          return dexFile != null ? dexFile.loadClassBinaryName(name, definingContext, suppressed) : null;
      }
  }
}

// DexFile
public final class DexFile {
  public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
      return defineClass(name, loader, mCookie, this, suppressed);
  }

  private static Class defineClass(String name, ClassLoader loader, Object cookie, DexFile dexFile, List<Throwable> suppressed) {
      Class result = null;
      try {
          result = defineClassNative(name, loader, cookie, dexFile);
      } catch (NoClassDefFoundError e) {
          if (suppressed != null) {
              suppressed.add(e);
          }
      } catch (ClassNotFoundException e) {
          if (suppressed != null) {
              suppressed.add(e);
          }
      }
      return result;
  }

  private static native Class defineClassNative(String name, ClassLoader loader, Object cookie, DexFile dexFile) throws ClassNotFoundException, NoClassDefFoundError;
}
```
具体来说，BaseDexClassLoader可以关联多个dex文件，每个dex文件会被封装成一个Element对象，这些Element对象会被排列成一个有序的dexElements数组。当BaseDexClassLoader收到一个类加载请求时，它会依次遍历dexElements数组进行查找，如果找到就直接返回，否则在下一个dex文件中继续查找。

因此，当两个类出现在不同的dex文件中时，排在前面的dex文件中的类会被优先加载，利用这一特点就可以实现Android热修复（热补丁）方案。所谓热修复，就是指应用能够在无需重新安装的情况实现更新，帮助应用快速建立动态修复能力的方案。
- Qzone技术团队早期就想到把修复好的之前有问题的类打包成一个patch.dex，然后把它插到dexElements数组的最前面，这样修复好的类就会优先被找到，从而替代掉当前版本中有问题的类。
- 微信Tinker方法也是类似的思想，不过区别在于它采用的是全量Dex替换。具体来说，首先通过比较新apk中的dex文件和旧apk中的dex文件得到差异并生成补丁包patch.dex，然后将其下发给客户端，客户端会将patch.dex和原始安装包中的旧dex合并生成新的dex文件，最后通过反射将之放置到dexElements数组的最前面。Tinker这样做的原因是为了解决Qzone方案遇到的[问题](https://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a)。

参考资料：
1. [深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）](https://item.jd.com/12607299.html)
2. [Dalvik 和 ART 有什么区别？深扒 Android 虚拟机发展史，真相却出乎意料！](https://juejin.im/post/5c232907f265da61662482b4)
