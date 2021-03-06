# Java垃圾收集器与内存分配策略
Java技术体系的自动内存管理，最根本的目标是自动化地解决了两个问题：自动给对象分配内存以及自动回收分配给对象的内存。
## Java内存区域
JVM在执行Java程序的过程中会把它所管理的内存划分为若干个不同的数据区域。这些区域有各自的用途，以及创建和销毁的时间，有的区域随着虚拟机进程的启动而一直存在，有些区域则是依赖用户线程的启动和结束而建立和销毁。

### 1. 程序计数器
程序计数器是当前线程所执行的字节码的行号指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是本地（Native）方法，这个计数器值则应为空。

### 2. Java虚拟机栈
与程序计数器一样，Java虚拟机栈也是线程私有的，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的线程内存模型：每个方法被执行的时候，JVM都会同步创建一个栈帧用于存储局部变量表、操作数栈、动态连接、方法出口等信息。每一个方法被调用直到执行完毕的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。

这个内存区域规定了两类异常状况：如果线程请求的栈深度大于JVM所允许的深度，将抛出`StackOverFlowError`异常；如果JVM栈容量可以动态扩展，当栈扩展时无法申请到足够的内存会抛出`OutOfMemoryError`异常。

### 3. 本地方法栈
作用与虚拟机栈相似，它们之间的区别是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的本地（Native）方法服务。与虚拟机栈一样，本地方法栈也会在栈深度溢出或栈扩展失败时分别抛出`StackOverFlowError`和`OutOfMemoryError`异常。

### 4. Java堆
Java堆是被所有**线程共享**的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，Java世界里“几乎”所有的对象实例都是在这里分配内存。Java堆是垃圾收集器管理的内存区域，

Java堆既可以被实现成固定大小的，也可以是可扩展的，不过当前主流的Java虚拟机都是按照可扩展来实现的（通过参数-Xmx和-Xms设定）。如果在Java堆中没有内存完成实例分配，并且堆也无法再扩展时，JVM将会抛出`OutOfMemoryError`异常。

### 5. 方法区
与Java堆一样，方法也是各个线程共享的内存区域，它用于存储已被虚拟机加载的**类型信息**、**常量**、**静态变量**、**即时编译器编译后的代码缓存**等数据。

### 6. 运行时常量池
运行时常量池是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池表，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中。当常量池无法再申请到内存时也会抛出OutOfMemoryError异常。

### 7. 直接内存
其实，直接内存并不是虚拟机运行时数据区的一部分，也不是《Java虚拟机规范》中定义的内存区域。但是这部分内存区域也被频繁地使用，而且也可能导致OutOfMemoryError异常出现，所以可以一起分析一下。

在JDK 1.4中新加入地NIO类，引入了一种基于通道与缓冲区的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作。这样就能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。

虽然本机直接内存的分配不会受到Java堆大小的限制，但是肯定还是会受到本机总内存大小和处理器寻址空间的限制。一般在配置虚拟机参数时，会根据实际内存去设置-Xmx等参数信息，但经常会忽略直接内存，使得各个内存区域总和大于物理内存限制，从而导致动态扩展时出现OutOfMemoryError异常。

## 垃圾收集
Java堆和方法区这两个区域有着很显著的不确定性：一个接口的多个实现类需要的内存可能会不一样，一个方法所执行的不同条件分支所需要的内存也可能不一样，只有处于运行期间，我们才能知道程序究竟会创建哪些对象，创建多少个对象，这部分内存的分配和回收是动态的。垃圾收集器所关注的正是这部分内存该如何管理。

### 判断对象是否存活
垃圾收集器在对堆进行回收之前，第一件事情就是要确定这些对象中哪些还“存活”着，哪些已经“死去”（即不可能再被任何途径使用的对象）了。

#### 1. 引用计数法
在对象中添加一个引用计数器，每当有一个地方引用它时，引用计数器值就加一；当引用失效时，计数器值就减一；任何时刻计数器值为零就是不可能再被使用的。

该算法虽然简单，但是有很大的局限性，例如很难解决对象之间相互循环引用的问题。

#### 2. 可达性分析法
通过一系列称为“GC Roots”的根对象作为起始节点集，从这些节点开始，根据引用关系向下搜索，搜索过程所走过的路径称为“引用链”，如果某个对象到GC Roots间没有任何引用链，或者用图论的话来说就是从GC Roots到这个对象不可达时，则证明此对象是不可能再被使用的。

在Java技术体系里，固定可作为GC Roots的对象包括以下几种：
- 在虚拟机栈（栈帧中的本地变量表）中引用的对象，譬如各个线程被调用的方法堆栈中使用到的参数、局部变量、临时变量等
- 在方法区中类静态属性引用的对象，譬如Java类的引用类型静态变量
- 在方法区中常量引用的对象，譬如字符串常量池里的引用
- 在本地方法栈中JNI（即通常所说的native方法）引用的对象
- Java虚拟机内部的引用，如基本数据类型队对应的Class对象，一些常驻的异常对象（比如NullPointException、OutOfMemoryError）等，还有系统类加载器
- 所有被同步锁（synchronized关键字）持有的对象
- 反映JVM内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等

#### 3. 引用
- **强引用**是最传统的“引用”的定义，是指在程序代码之中普遍存在的引用赋值，即类似“Object obj = new Object()”这种引用关系。无论任何情况下，只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象。
- **软引用**是用来描述一些还有用，但并非必须的对象。只要被软引用关联着的对象，在系统将要发生内存溢出异常时，会把这些对象列进回收范围之中进行第二次回收，如果这次回收还没有足够的内存，才会抛出内存溢出异常。
- **弱引用**也是用来描述那些非必须对象，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生为止。当垃圾收集器开始工作，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。
- **虚引用**也称为“幽灵引用”或者“幻影”引用，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的只是为了能在这个对象被收集器回收时收到一个系统通知。

#### 4. 回收方法区
《Java虚拟机规范》提到过可以不要求虚拟机在方法区中实现垃圾收集，因为方法区垃圾收集的“性价比”通常比较低：在Java堆中，尤其是在新生代中，对常规应用进行一次垃圾收集通常可以回收70%~99%的内存空间，相比之下，方法区回收囿于苛刻的判定条件，其区域垃圾收集的回收成果往往远低于此。

方法区的回收主要回收两部分内容：废弃的常量和不再使用的类型。

### 垃圾收集算法
当前商业虚拟机的垃圾收集器，大多数都遵循了“分代收集”的理论进行设计，它建立在三个分代假说之上：
- 弱分代假说：绝大多数对象都是朝生夕灭的。
- 强分代假说：熬过越多次垃圾收集过程的对象就越难以消亡。
- 跨代引用假说：跨代引用相对于同代引用来说仅占极少数。

所以收集器应该将Java堆划分出不同的区域，然后将回收对象根据其年龄（年龄即对象熬过垃圾收集过程的次数）分配到不同的区域之中存储。显而易见，如果一个区域中大多数对象都是朝生夕灭，难以熬过垃圾收集过程的话，那么把它们集中放在一起，每次回收时只关注如何保留少量存活而不是去标记那些大量将要被回收的对象，就能以较低代价回收到大量空间；如果剩下的都是难以消亡的对象，那把它们集中放在一起，虚拟机便可以使用较低的频率来回收这个区域，这就同时兼顾了垃圾收集的时间开销和内存空间的有效利用。

根据回收的目标区域不同，垃圾收集可分为以下几类：
- 新生代收集（Minor GC/Young GC）：目标只是新生代的垃圾收集
- 老年代收集（Major GC/Old GC）：目标只是老年代的垃圾收集
- 混合收集（Mixed GC）：目标是收集整个新生代以及部分老年代的垃圾收集
- 整堆收集（Full GC）：目标是收集整个Java堆和方法区的垃圾收集

#### 1. 标记-清除算法
分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后，统一回收所有被标记的对象。（也可以反过来，标记存活的对象，统一回收掉所有未被标记的对象。）

缺点：① 执行效率不稳定，如果Java堆中包含大量对象，而且其中大部分是需要被回收的，这时必须进行大量标记和清除的动作，导致标记和清除两个过程的执行效率都随对象数量增长而降低；② 内存空间的碎片化问题，标记、清除后会产生大量不连续的内存碎片，空间碎片太多可能会导致当以后程序运行过程中需要分配较大对象时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

#### 2. 标记-复制算法
半区复制：将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块内存快用完了，就将还存活着的对象复制到另一块内存上，然后再把已使用过的内存空间一次清理掉。

缺点：① 如果内存中大多数对象都是存活的，这种算法将会产生大量的内存间复制开销；② 可用内存缩小为原来的一半，比较浪费。

优化的半区复制策略，“Appel式回收”：把新生代分为一块较大的Eden空间和两块较小的Survivor空间，每次分配内存只使用Eden和其中一块Survivor空间。发生垃圾收集时，将Eden和Survivor中仍然存活的对象一次性复制到另外一块Survivor空间上，然后直接清理掉Eden和已用过的那块Survivor空间。当Survivor空间不足以容纳一次Minor GC之后存活的对象时，就需要依赖其他内存区域（实际上大多就是老年代）进行分配担保。

#### 3. 标记-整理算法
标记过程仍然与“标记-清除”算法一样，但是后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都想内存空间的一端移动，然后直接清理掉边界以外的内存。所以，标记-清除算法和标记-整理算法的本质差异在于前者是一种非移动式的回收算法，而后者是移动式的。

缺点：在都有大量对象存活的区域，尤其是老年代，移动存活对象并更新所有引用这些对象的地方将会是一种极为负重的操作，而且这种对象移动操作必须全程暂停用户应用程序才能进行。这就是所谓的“Stop the world”问题。

是否移动对象都存在弊端，需要使用者进行权衡：移动则内存回收会更复杂了，不移动则内存分配时会更复杂。

