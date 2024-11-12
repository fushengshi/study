# Java虚拟机（JVM）

## 内存区域

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240510002759034.png" alt="image-20240510002759034" style="zoom: 80%;" />

线程共享的：堆、方法区、直接内存（非运行时数据区的一部分）。

线程私有的：程序计数器、虚拟机栈、本地方法栈。

运行时常量池、方法区、字符串常量池这些都是不随虚拟机实现而改变的逻辑概念，是公共且抽象的，Metaspace、Heap 是与具体某种虚拟机实现相关的物理概念，是私有且具体的。

### 堆

Java 虚拟机所管理的内存中最大的一块，Java 堆是所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。

#### 字符串常量池

字符串常量池 是 JVM 为了提升性能和减少内存消耗针对字符串（String 类）专门开辟的一块区域，主要目的是为了避免字符串的重复创建。

> Java 7 之前，字符串常量池存放在永久代，Java 7 及之后字符串常量池和静态变量从永久代移动了 Java 堆中。
>
> 主要是因为永久代（方法区实现）的 GC 回收效率太低，只有在整堆收集 Full GC 的时候才会被执行 GC。Java 程序中通常会有大量的被创建的字符串等待回收，将字符串常量池放到堆中，能够更高效及时地回收字符串内存。

#### 静态变量

`static` 关键字修饰的变量，只与类相关也称为类变量。

> Java 7 及之后 HotSpot 已经把原本放在永久代的字符串常量池、静态变量等移动到堆中，类变量随着 Class 对象一起存放在 Java 堆中。



### 方法区

方法区属于是 JVM 运行时数据区域的一块逻辑区域，是各个线程共享的内存区域。

当虚拟机要使用一个类时，需要读取并解析 Class 文件获取相关信息，再将信息存入到方法区。方法区会存储已被虚拟机加载的**类信息、字段信息、方法信息、常量、即时编译器编译后的代码缓存等数据**。

永久代以及元空间是 HotSpot 虚拟机对虚拟机规范中方法区的两种实现方式。元空间与永久代区别：元空间不在虚拟机中，使用的本地内存，默认情况下，元空间的大小仅受本地内存限制。

#### 运行时常量池

Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有用于存放编译期生成的各种字面量（Literal）和符号引用（Symbolic Reference）的常量池表（Constant Pool Table）。



### 程序计数器

程序计数器（Program Counter Register）是一块较小的内存空间，是当前线程所执行的字节码的行号指示器。字节码解释器工作时通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等功能都需要依赖这个计数器来完成。

- 字节码解释器通过改变程序计数器来依次读取指令，从而实现代码的流程控制，如：顺序执行、选择、循环、异常处理。
- 在多线程的情况下，程序计数器用于记录当前线程执行的位置，从而当线程被切换回来的时候能够知道该线程上次运行到哪了。

> JVM 中的程序计数器（PC寄存器），Register 命名源于 CPU 的寄存器，寄存器存储指令相关的现场信息，CPU 只有把数据装载到寄存器才能够运行。
>
> JVM 中的程序计数器是对物理 PC 寄存器的一种**抽象模拟**。执行 Java 方法时，这个抽象的 PC 寄存器存的是 Java 字节码的地址。
>
> 每个 Java 线程都直接映射到一个 OS 线程上执行。native 方法由原生平台直接执行，并不理会抽象的 JVM 层面上的 PC 寄存器概念，原生的 CPU 上真正的 PC 寄存器是怎样就是怎样。因此 Java 多线程执行 native 方法时程序计数器为空。



### 虚拟机栈

栈由一个个栈帧组成，而每个栈帧中都拥有：局部变量表、操作数栈、动态链接、方法返回地址。

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240511021042342.png" alt="image-20240511021042342" style="zoom:67%;" />

#### 局部变量表

主要存放了编译期可知的各种基本数据类型、对象引用（reference 类型，它不同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）。

#### 操作数栈

主要作为方法调用的中转站使用，用于存放方法执行过程中产生的中间计算结果。另外，计算过程中产生的临时变量也会放在操作数栈中。

#### 动态链接

主要服务一个方法需要调用其他方法的场景。Class 文件的常量池里保存有大量的符号引用比如方法引用的符号引用。

当一个方法要调用其他方法，需要将**运行时常量池**中指向方法的符号引用转化为其在内存地址中的直接引用。动态链接的作用就是为了将符号引用转换为调用方法的直接引用，这个过程也被称为动态连接 。



### 本地方法栈

虚拟机栈为虚拟机执行 Java 方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的 native 方法服务。

本地方法被执行时，在本地方法栈也会创建一个栈帧，用于存放该本地方法的局部变量表、操作数栈、动态链接、出口信息。

> 在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。



### 直接内存

直接内存是一种特殊的内存缓冲区，并不在 Java 堆或方法区中分配的，而是通过 JNI 的方式在本地内存上分配的。

NIO（Non-Blocking I/O，New I/O）引入了一种基于通道（Channel）与缓存区（Buffer）的 I/O 方式，直接使用 native 函数库直接分配堆外内存，然后通过一个存储在 Java 堆中的 DirectByteBuffer 对象作为这块内存的引用进行操作。

> 详见网络系统。

直接内存回收：

1. 业务代码 将 DirectByteBuffer 置为 null，表示想要回收这块指向的堆外内存。
2. JVM 垃圾回收器检测到该 DirectByteBuffer 对象不可达，将其回收，然后将它对应的虚引用对象 Cleaner 放到 Reference 的 pending 属性中。
3. 后台守护线程 ReferenceHandler 执行 `tryHandlePending()` 方法。检测到 pending 属性不为空，则拿到 Cleaner 对象，然后调用 Cleaner 对象的 `clean()` 方法。在 Cleaner 对象的 `clean()` 方法中，会调用 DirectByteBuffer 的内部类 Deallocator 的 `run()` 方法。在 `run()` 方法中，会调用 `unsafe.freeMemory()` 方法，从而释放了堆外内存。



## JVM 垃圾回收

堆内存逻辑：

- 新生代

  新生代（Young Generation Space，PSYoungGen），默认堆内存 1/3。

  分为伊甸区（Eden）和幸存者区（Form、To），默认 8:1:1。

- 老年代

  老年代（Tenure Generation Space，ParOldGen），默认堆内存 2/3。

GC 分为两大类，分别是 Partial GC 和 Full GC。

- 部分收集（Partial GC）

  - 新生代收集（Minor GC / Young GC）

    只对新生代进行垃圾收集。

  - 老年代收集（Major GC / Old GC）

    只对老年代进行垃圾收集。需要注意的是 Major GC 在有的语境中也用于指代整堆收集。

  - 混合收集（Mixed GC）

    对整个新生代和部分老年代进行垃圾收集。

- 整堆收集（Full GC）

  收集整个 Java 堆（包括年轻代、老年代）和方法区。

### GC 触发条件

#### Young GC

- Eden 快要被占满时会触发，包括对象分配内存不够或者为 TLAB 分配内存不够。

  > TLAB（Thread Local Allocation Buffer）
  >
  > 为每一个线程预先在 Eden 区分配一块儿内存，JVM 在给线程中的对象分配内存时首先在 TLAB 分配，当对象大于 TLAB 中的剩余内存或 TLAB 的内存已用尽时，再采用 CAS 进行内存分配。

- 一些收集器的回收实现在 Full GC 前先执行 Young GC。

#### Full GC

- 将要进行 Young GC 时，根据之前统计数据发现年轻代平均晋升大小比老年代剩余空间大，会触发 Full GC。

- 分配担保机制（Promotion Failure）

  当年轻代向老年代晋升对象时，老年代没有足够的空间容纳这些对象，会触发 Full GC。

  - 新生代的 To 区放不下从 Eden 和 From 区拷贝过来对象。
  - 新生代对象 GC 年龄到达阈值。

  > PLAB（Promotion Local Allocation Buffers）
  >
  > 多线程并行执行 Young GC 时，可能有很多对象需要晋升到老年代。先从老年代 freelist（空闲链表）申请一块空间，然后在这一块空间中就可以通过指针碰撞（bump the pointer）来分配内存。每个线程先申请一块作为 PLAB ，然后在这一块内存里面分配晋升的对象。

- 大对象直接在老年代申请分配，老年代空间不足则会触发 Full GC。

- 永久代或元空间不足时会触发 Full GC。

- 执行 `System.gc()`、`jmap -dump` 等命令会触发 Full GC。

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240512010808327.png" alt="image-20240512010808327" style="zoom:67%;" />

> SLAB 分配器通过预先分配一块大的内存区域，把这块内存区域划分为多个大小相等的块，然后把对象分配在这些块上，这样就可以减少内存碎片的问题。



### 可达性分析算法

#### GC Roots

- 虚拟机栈（栈帧中的局部变量表）中引用的对象。
- 本地方法栈（native 方法）中引用的对象。
- 方法区中类静态属性引用的对象。
- 方法区中常量引用的对象。
- 所有被同步锁持有的对象。
- JNI（Java Native Interface）引用的对象。

#### 三色标记法

要找出存活对象，根据可达性分析，从 GC Roots 开始进行遍历访问，可达的则为存活对象。

把遍历对象图过程中遇到的对象，按是否访问过这个条件标记成以下三种颜色：

- 白色：尚未访问过。
- 黑色：本对象已访问过，而且本对象引用 的其他对象 也全部访问过了。
- 灰色：本对象已访问过，但是本对象引用到的其他对象尚未全部访问完。全部访问后，会转换为黑色。

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240511011021903.png" alt="image-20240511011021903" style="zoom:80%;" />



##### 漏标

- 赋值器插入了一条或者多条**从黑色对象**到白色对象的新引用。

- 赋值器删除了全部从灰色对象到该白色对象的直接或间接引用。

解决方案：

- 增量更新（Incremental Update）：

  当黑色指向白色的引用被建立时，就将这个**新的引用关系记录下来**，等扫描结束后，再以这些记录中的黑色对象为根，重新扫描一次。相当于黑色对象一旦建立了指向白色对象的引用，就会变为灰色对象。

  > 卡表只有一个，每次 Young GC 都会扫描重置卡表，增量更新的记录就会清理。定义了 mod-union table，在并发标记时如果发生 Young GC 重置卡表的记录时，会更新  mod-union table 对应的位置。
  >
  > CMS 重新标记阶段可以结合卡表和  mod-union table 处理增量更新，防止漏标对象。

- 原始快照（Snapshot At The Beginning，SATB）：

  当对象的成员变量的引用发生变化时，可以利用写屏障，**将原来成员变量的引用对象记录下来**。

  尝试保留开始时的对象图，即原始快照，当某个时刻的 GC Roots 确定后，当时的对象图就已经确定了。

  > 写屏障是指在赋值操作前后，加入一些处理（可以参考 AOP 的概念）。

#### RSet

RSet（记忆集、RememberSet）记录老年代中的新生代的引用的对象地址。在进行标记时，除了从 GC Roots 开始遍历，还会从 RSet 遍历，确保标记该区域所有存活的对象。

- CMS 的记忆集的实现是卡表。

  卡表（Card Table）是 Hotspot 中记忆集的具体实现，通过写后屏障维护。卡表中的每一个元素都对应着一块特定大小的内存块卡页（card page），当存在跨代引用的时候将卡页标记为脏。Young GC 扫描时会扫描老年代的卡表，避免全堆扫描。

  > 支持 Young GC 避免全堆扫描。
  >
  > 支持 CMS 增量更新（结合 mod-union table，见增量更新）。

- G1 的记忆集是 hash table。

  G1 是基于 Region 的，所以在 points-out 的卡表之上还加了个 points-into 的结构。因为一个 Region 需要知道有哪些别的 Region 有指向自己的指针，还需要知道这些指针在哪些 card 中。G1 的记忆集 hash table，key 是别的 Region 的起始地址，value 是一个集合存储 card table 的 index。

  > G1 除了像传统垃圾回收器那样记录老年代到新生代的引用之外，还需要额外记录老年代到老年代的引用关系。



### 垃圾收集算法

#### 复制算法

为了解决标记-清除算法的效率和内存碎片问题，复制（Copying）收集算法出现了。它可以将内存分为大小相同的两块，每次使用其中的一块。当这一块的内存使用完后，就将还存活的对象复制到另一块去，然后再把使用的空间一次清理掉。这样就使每次的内存回收都是对内存区间的一半进行回收。

#### 标记-清除算法

标记-清除（Mark-and-Sweep）算法分为标记（Mark）和清除（Sweep）阶段，首先标记出所有不需要回收的对象，在标记完成后统一回收掉所有没有被标记的对象。

#### 标记-整理算法

标记-整理（Mark-and-Compact）算法是根据老年代的特点提出的一种标记算法，标记过程仍然与 标记-清除 算法一样，但后续步骤不是直接对可回收对象回收，而是让所有存活的对象向一端移动，然后直接清理掉端边界以外的内存。

#### 分代收集算法

当前虚拟机的垃圾收集都采用分代收集算法，这种算法没有什么新的思想，只是根据对象存活周期的不同将内存分为几块。一般将 Java 堆分为新生代和老年代，可以根据各个年代的特点选择合适的垃圾收集算法。

比如在新生代中，每次收集都会有大量对象死去，所以可以选择”标记-复制“算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。而老年代的对象存活几率是比较高的，而且没有额外的空间进行分配担保，所以必须选择 标记-清除 或 标记-整理 算法进行垃圾收集。



### 垃圾收集器

#### Parallel Scavenge 收集器

新生代采用标记-复制算法，老年代采用标记-整理算法。

Parallel Scavenge 收集器关注点是吞吐量（高效率的利用 CPU）。CMS 等垃圾收集器的关注点更多的是用户线程的停顿时间（提高用户体验）。所谓吞吐量就是 CPU 中用于运行用户代码的时间与 CPU 总消耗时间的比值。

> Java 8 默认使用的是 Parallel Scavenge + Parallel Old。

#### Parallel Old 收集器

Parallel Scavenge 收集器的老年代版本。使用多线程和 标记-整理 算法。在注重吞吐量以及 CPU 资源的场合，都可以优先考虑 Parallel Scavenge 收集器和 Parallel Old 收集器。

#### CMS 收集器

CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。

CMS（Concurrent Mark Sweep）收集器是 HotSpot 虚拟机第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。

- 初始标记

  STW（Stop The World），并记录下直接与 root 相连的对象，速度很快。

- 并发标记

  同时开启 GC 和用户线程，用一个闭包结构去记录可达对象。因为用户线程可能会不断的更新引用域，所以 GC 线程无法保证可达性分析的实时性。所以这个算法里会跟踪记录这些发生引用更新的地方。

- 重新标记

  STW，重新标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录。

  重新遍历一遍新生代对象、GC Roots、卡表等来修正标记。

- 并发清除

  开启用户线程，同时 GC 线程开始对未标记的区域做清扫。

> CMS 垃圾回收器在 Java 9 中已经被标记为过时，并在 Java 14 中被移除。

#### G1 收集器

G1（Garbage-First）是一款面向服务器的垃圾收集器，主要针对配备多颗处理器及大容量内存的机器. 以极高概率满足 GC 停顿时间要求的同时,还具备高吞吐量性能特征。

区域划分和优先级区域回收机制，确保 G1 收集器可以在有限时间获得最高的垃圾收集效率。

- G1 收集器避免全区域垃圾收集，把堆内存划分为大小固定的几个独立区域，并且跟踪这些区域的垃圾收集进度。
- 在后台维护一个优先级列表，每次根据所允许的收集时间，优先回收垃圾最多的区域。

回收步骤：

- 初始标记

  STW，仅使用一条初始标记线程对 GC Roots 关联的对象进行标记。在 G1 中标记对象是利用外部的 bitmap 来记录，而不是对象头。

- 并发标记

  使用一条标记线程与用户线程并发执行。此过程进行可达性分析，速度很慢。STAB 也会在这个阶段记录着变更的引用。

- 最终标记

  STW，使用多条标记线程并发执行。处理 STAB 中的引用。

- 筛选回收

  STW，使用多条筛选回收线程并发执行。根据标记的 bitmap 统计每个 Region 存活对象的多少，如果有完全没存活的 Region 则整体回收。

RSet（记忆集）记录不同分代之间的引用关系，因为 G1 将内存划分为若干的 Region，在局部回收时，跨区引用的问题要比传统的垃圾回收器复杂。除了像传统垃圾回收器那样记录老年代到新生代的引用之外，还需要额外记录老年代到老年代的引用关系。

> 从 Java 9 开始，G1 垃圾收集器成为了默认的垃圾收集器。



## 对象创建

### 类加载检查

虚拟机遇到一条 new 指令时，首先将去检查这个指令的参数是否能在常量池中定位到这个类的符号引用，并且检查这个符号引用代表的类是否已被加载过、解析和初始化过。如果没有必须先执行相应的**类加载**过程。

> 详见类加载。

### 分配内存

为新生对象分配内存，对象所需的内存大小在类加载完成后便可确定，为对象分配空间的任务等同于把一块确定大小的内存从 Java 堆中划分出来。

#### 分配内存方式

- 指针碰撞

  适用场合：堆内存规整（没有内存碎片）的情况下。

  原理：用过的内存全部整合到一边，没有用过的内存放在另一边，中间有一个分界指针，只需要向着没用过的内存方向将该指针移动对象内存大小位置即可。

  使用该分配方式的 GC 收集器：Serial，ParNew。

- 空闲列表

  适用场合：堆内存不规整的情况下。

  原理：虚拟机会维护一个列表，该列表中会记录哪些内存块是可用的，在分配的时候，找一块儿足够大的内存块儿来划分给对象实例，最后更新列表记录。

  使用该分配方式的 GC 收集器：CMS。

#### 内存分配并发问题

在创建对象的时候线程安全很重要，在实际开发过程中创建对象是很频繁的事情，作为虚拟机必须要保证线程是安全的，通常虚拟机采用两种方式来保证线程安全：

- CAS + 失败重试

  CAS 是乐观锁的一种实现方式。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。**虚拟机采用 CAS 配上失败重试的方式保证更新操作的原子性。**

- TLAB（Thread Local Allocation Buffer）

  为每一个线程预先在 Eden 区分配一块儿内存，JVM 在给线程中的对象分配内存时首先在 TLAB 分配，当对象大于 TLAB 中的剩余内存或 TLAB 的内存已用尽时，再采用 CAS 进行内存分配。

  > TLAB 是 JVM 中的一种内存区域，它为每个线程分配独立的内存空间，用于存储线程私有的对象实例和本地数据。

### 初始化零值

内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头），这一步操作保证了对象的实例字段在 Java 代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。

### 设置对象头

初始化零值完成之后，虚拟机要对对象进行必要的设置，这些信息存放在对象头中。

> 对象头见附录。

### 执行 init 方法

上面工作都完成之后，从 JVM 视角一个新的对象已经产生，但从 Java 程序的视角来看，对象创建才刚开始，`<init>` 方法还没有执行，所有的字段都还为零。一般执行 new 指令之后会接着执行 `<init>` 方法，把对象进行初始化，一个真正可用的对象产生。



## 类加载

系统加载 Class 类型的文件主要三步：加载、连接（验证、准备、解析）、初始化。

### 加载

一个非数组类的加载阶段（加载阶段获取类的二进制字节流的动作）是可控性最强的阶段，这一步可以自定义类加载器去控制字节流的获取方式（重写一个类加载器的 `loadClass()` 方法）。

> 数组类不是通过 ClassLoader 创建的，而是 JVM 在需要的时候自动创建的，数组类通过 `getClassLoader()` 方法获取 ClassLoader 的时候和该数组的元素类型的 ClassLoader 是一致的。

#### 类加载器

类加载器的主要作用就是加载 Java 类的字节码（.class 文件）到 JVM 中（在内存中生成一个代表该类的 Class 对象）。

JVM 启动的时候，并不会一次性加载所有的类，而是根据需要去动态加载。也就是说，大部分类在具体用到的时候才会去加载，这样对内存更加友好。对于已经加载的类会被放在 ClassLoader 中。在类加载的时候，系统会首先判断当前类是否被加载过。已经被加载的类会直接返回，否则才会尝试加载。

三个重要的 ClassLoader：

- BootstrapClassLoader（启动类加载器）

  最顶层的加载类，由 C++实现，通常表示为 null 并且没有父级，主要用来加载 JDK 内部的核心类库（`%JAVA_HOME%/lib` 目录下的 rt.jar、resources.jar、charsets.jar 等 jar 包和类）以及被 `-Xbootclasspath` 参数指定的路径下的所有类。

  > BootStrapClassLoader 是用 C++ 写的，`getClassLoader()` 返回 BootStrapClassLoader 时会返回 null。

- ExtensionClassLoader（扩展类加载器）

  主要负责加载 `%JRE_HOME%/lib/ext` 目录下的 jar 包和类以及被 `java.ext.dirs` 系统变量所指定的路径下的所有类。

- AppClassLoader（应用程序类加载器）

  面向我们用户的加载器，负责加载当前应用 classpath 下的所有 jar 包和类。

除了 BootstrapClassLoader 是 JVM 自身的一部分之外，其他所有的类加载器都是在 JVM 外部实现的，并且全都继承自 `ClassLoader`抽象类。用户可以自定义类加载器，以便让应用程序自己决定如何去获取所需的类。

#### 双亲委派机制

每当一个类加载器接收到加载请求时，它会先将请求转发给父类加载器。在父类加载器没有找到所请求的类的情况下，该类加载器才会尝试去加载。

双亲委派模型保证了 Java 程序的稳定运行，可以避免类的重复加载（JVM 区分不同类的方式不仅仅根据类名，相同的类文件被不同的类加载器加载产生的是两个不同的类），也保证了 Java 的核心 API 不被篡改。

自定义加载器需要继承 ClassLoader 。

- 如果不想打破双亲委派模型，重写 ClassLoader 类中的 `findClass()` 方法，无法被父类加载器加载的类最终会通过这个方法被加载。
- 如果想打破双亲委派模型重写 `loadClass()` 方法。

> Tomcat 服务器为了能够优先加载 Web 应用目录下的类，然后再加载其他目录下的类，就自定义了类加载器 WebAppClassLoader 来打破双亲委托机制。这也是 Tomcat 下 Web 应用之间的类实现隔离的具体原理。

### 验证

确保 Class 文件的字节流中包含的信息符合《Java 虚拟机规范》的全部约束要求，保证这些信息被当作代码运行后不会危害虚拟机自身的安全。

验证阶段主要由四个检验阶段组成：

- 文件格式验证（Class 文件格式检查）

- 元数据验证（字节码语义检查）

- 字节码验证（程序语义检查）

- 符号引用验证（类的正确性检查）

  符号引用验证发生在类加载过程中的**解析阶段**，具体在 JVM 将符号引用转化为直接引用的时候。

### 准备

- 静态变量分配内存。
- 设置静态变量初始值。

进行内存分配的仅包括静态变量（类变量，static 关键字修饰），而不包括实例变量。实例变量会在对象实例化时随着对象一块分配在 Java 堆中。

> 在 Java 7 及之后，HotSpot 已经把原本放在永久代的字符串常量池、静态变量等移动到堆中，静态变量随着 Class 对象一起存放在 Java 堆中。

### 解析

- 通过方法表将常量池内的符号引用替换为直接引用。

  Java 虚拟机为每个类都准备了一张方法表来存放类中所有的方法。当需要调用一个类的方法的时候，只要知道这个方法在方法表中的偏移量就可以直接调用该方法了。

  通过解析操作符号引用就可以直接转变为目标方法在类中方法表的位置，从而使方法可以调用。

### 初始化

初始化阶段是执行初始化方法 `<clinit>` 方法的过程，是类加载的最后一步， JVM 开始真正执行类中定义的 Java 程序代码（字节码）。

> Java 编译器会收集所有的类变量初始化语句和类型的静态初始化器，放到 `<clinit>`  方法中。
>
> `<clinit>` 方法是编译之后自动生成的。对于 `<clinit>` 方法的调用，虚拟机会自己确保其在多线程环境中的安全性。因为 `<clinit>` 方法是带锁线程安全，所以在多线程环境下进行类初始化的话可能会引起多个线程阻塞，并且这种阻塞很难被发现。

- 当遇到 `new`、 `getstatic`、`putstatic` 或 `invokestatic` 这 4 条字节码指令时

  - 当 JVM 执行 `new` 指令时会初始化类。即当程序创建一个类的实例对象。
  - 当 JVM 执行 `getstatic` 指令时会初始化类。即程序访问类的静态变量（不是静态常量，常量会被加载到运行时常量池）。
  - 当 JVM 执行 `putstatic` 指令时会初始化类。即程序给类的静态变量赋值。
  - 当 JVM 执行 `invokestatic` 指令时会初始化类。即程序调用类的静态方法。

- 使用 java.lang.reflect 包的方法对类进行反射调用时

  如 `Class.forName("")`、`newInstance()` 等等，如果类没初始化，需要触发其初始化。

  >`TargetObject.class`、`ClassLoader.getSystemClassLoader().loadClass("")` 不会进行初始化。
  >
  >不进行包括初始化等一系列步骤，静态代码块和静态对象不会得到执行。

- 初始化一个类，如果其父类还未初始化，则**先触发该父类的初始化**。

- 当虚拟机启动时，用户需要定义一个要执行的主类（包含 main 方法的那个类），虚拟机会先初始化这个类。

- `MethodHandle` 和 `VarHandle` 可以看作是轻量级的反射调用机制，而要想使用这 2 个调用， 就必须先使用 `findStaticVarHandle()` 来初始化要调用的类。

- 当一个接口中定义了 default 接口方法时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。

>static 静态代码块的执行在初始化阶段， JVM 主要完成对静态变量的初始化，静态代码块执行等工作。



# 附录

## 打破双亲委派

### Tomcat 打破双亲委派

Tomcat 通过 `loadClass()` 打破双亲委派。

Tomcat 作为一个 Web 容器应该具有一下特点：

- 隔离性：Web 应用类库应该相互隔离，避免依赖库或者应用包互相影响。
- 灵活性：Web 应用支持热加载，热部署。

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240605234511848.png" alt="image-20240605234511848" style="zoom:67%;" />

Tomcat 的设计是违背 Java 的双亲委派模型的，每个 WebappClassLoader 加载自己目录下的 .class 文件，不会传递给父加载器，打破了双亲委派机制，实现隔离性。

类加载器流程：

- 从内存缓存中加载。

- 如果没有，用 ExtClassLoader 扩展类加载器加载。

  ExtClassLoader 会委托给 BootstrapClassLoader 去加载，JRE 里的类由 BootstrapClassLoader 安全加载。

  防止 Web 应用自己的类覆盖 JRE 的核心类。

- 如果没有，则从当前类加载器 WebappClassLoader 加载（按照 WEB-INF/classes、WEB-INF/lib 的顺序）。

- 如果没有，则从父类加载器加载，父类加载器采用默认的委派模式，由系统类加载器加载。

**热加载：**

RestartClassLoader 为自定义的类加载器，其核心是 `loadClass()` 的加载方式，SpringBoot devtools 中修改了双亲委托机制，默认优先从自己加载，如果自己没有加载到，则从 parent 进行加载。

保证了业务代码可以优先被 RestartClassLoader 加载，进而通过重新加载 RestartClassLoader 完成应用代码部分的重新加载。



### Spring 打破双亲委派

Fat Jar 中依赖的各个第三方 Jar 文件不在程序 classpath 下，如果采用双亲委派机制获取不到依赖的 Jar  包。org.springframework.boot.loader.LaunchedURLClassLoader 是 spring-boot-loader 中自定义的类加载器，实现对 jar 包中 BOOT-INF/classes 目录下的类和 BOOT-INF/lib 下第三方 jar 包中的类的加载。

SpringBoot 可执行 Jar 包的关键结构：

- BOOT-INF 目录

  包含项目代码（classes目录）以及所需要的依赖（lib 目录）。

- META-INF 目录

  通过 MANIFEST.MF 文件提供 Jar 包的元数据，声明了 Jar 的启动类。

- org.springframework.boot.loader

  Spring Boot 的加载器代码，实现 Jar in Jar 加载。

```java
protected void launch(String[] args) throws Exception {
	// 注册 URL（jar）协议的处理器
	JarFile.registerUrlProtocolHandler();
	// 先从archive（当前 jar 包应用）解析出所有的JarFileArchive，创建SpringBoot自定义的ClassLoader类加载器，可加载当前jar中所有的类
	ClassLoader classLoader = createClassLoader(getClassPathArchives());
	// 获取当前应用的启动类并执行
	launch(args, getMainClass(), classLoader);
}

protected void launch(String[] args, String mainClass, ClassLoader classLoader)
    throws Exception {
    // 将当前线程的上下文类加载器设置成 LaunchedURLClassLoader。
    Thread.currentThread().setContextClassLoader(classLoader);
    createMainMethodRunner(mainClass, args, classLoader).run();
}

public class LaunchedURLClassLoader extends URLClassLoader {
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        Handler.setUseFastConnectionExceptions(true);
        try {
            try {
                // 尝试根据类名去定义类所在的包
                definePackageIfNecessary(name);
            } catch (IllegalArgumentException ex) {
                if (getPackage(name) == null) {
                    throw new AssertionError("Package " + name + " has already been defined but it could not be found");
                }
            }
            // super.loadClass实际上会调用java.lang.ClassLoader#loadClass(java.lang.String, boolean)，
            // 遵循双亲委派机制进行查找类，而Bootstrap ClassLoader和Extension ClassLoader将会查找不到fat jar依赖的类，最终到Application ClassLoader，调用URLClassLoader#findClass
            return super.loadClass(name, resolve);
        } finally {
            Handler.setUseFastConnectionExceptions(false);
        }
    }
}
```

`launch()` 创建一个自定义的 ClassLoader 类加载器，主要可加载 BOOT-INF/classes 目录下的类，以及 BOOT-INF/lib 目录下的 jar 包中的类，然后调用 Spring Boot 应用启动类的 main 方法。

LaunchedURLClassLoader 重写了 ClassLoader 的 `loadClass()` 加载 Class 类对象方法，打破了双亲委派，在加载对应的 Class 类对象之前新增了一部分逻辑，会尝试从 jar 包中定义 Package 包对象，这样就能加载到对应的 Class 类对象。



### SPI 打破双亲委派

SPI 通过线程上下文类加载器 ThreadContextClassLoader 打破双亲委派。

在双亲委托模型下，类加载器是由下而上，即下层的类加载器会委托上层进行加载。对于 SPI 来说，有些接口是 Java 核心库所提供的，Java 核心库是由启动类加载器（BootstrapClassLoader）加载的，接口的实现却来自于不同厂商提供的 jar 包，Java 的启动类加载器不会加载其他来源的 jar 包，传统的双亲委托模型就无法满足 SPI 的要求。

通过给当前线程设置上下文类加载器，就可以由设置的上下文类加载器来实现对于接口实现类的加载。

- 如果创建线程时没有设置，则会从父线程中继承。
- 如果在应用程序的全局范围内都没有设置过，类加载器默认为 AppClassLoader。
- 父 ClassLoader 可以使用当前线程 `Thread.currentThread().getContextClassLoader()` 所指定的上下文类加载器加载类。

> BootStrapClassLoader 是用 C++ 写的，`getContextClassLoader()` 返回 BootStrapClassLoader 时会返回 null。

#### ContextClassLoader 在 JDBC 中的应用

JDBC 是 Java 提出的一个有关数据库访问和操作的一个标准，也就是定义了一系列接口。不同的数据库厂商（Oracle、MySQL、PostgreSQL等）提供对该接口的实现，即提供的 Driver 驱动包。

Java 定义的 JDBC 接口位于 JDK 的 `rt.jar`（java.sql）中，这些接口由 BootstrapClassLoader 进行加载。数据库厂商提供的 Driver 驱动包一般在应用程序中引入（比如位于 CLASSPATH 下）超出了 BootstrapClassLoader 的加载范围，这些驱动包中的 JDBC 接口的实现类无法通过 BootstrapClassLoader 加载，只能由 AppClassLoader 或自定义的 ClassLoader 加载。

```java
public class DriverManager { // Java 核心库
    static {
        // 在静态代码块中加载当前环境中的 JDBC Driver
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }

    private static void loadInitialDrivers() {
        ...
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
                // 通过ServiceLoader的load方法加载Driver的实现（如MySQL、Oracle提供的Driver实现）
                // SPI 机制
                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();

                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                }
                return null;
            }
        });
    }
}

public final class ServiceLoader<S> implements Iterable<S> {
    private static final String PREFIX = "META-INF/services/";

    public static <S> ServiceLoader<S> load(Class<S> service) {
        // 使用线程上下文的类加载器来创建 ServiceLoader
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
}
```



## SPI

SPI 服务提供接口（Service Provider Interface） 是 JDK 内置的一种服务提供发现机制，是 Java 提供的一套用来被第三方实现或者扩展的 API，它可以用来启用框架扩展和替换组件（可通过 SPI 机制实现模块化）。Java 的 SPI 机制可以为某个接口寻找服务实现。SPI 机制主要思想是将装配的控制权移到程序之外，在模块化设计中这个机制尤其重要，其核心思想就是解耦。

SPI 机制的具体实现本质上还是通过反射完成的。按照规定将要暴露对外使用的具体实现类在 `META-INF/services/` 文件下声明。

<img src="https://gitee.com/fushengshi/image/raw/master/image-20240508221907339.png" alt="image-20240508221907339" style="zoom: 80%;" />

JDK SPI 的入口方法是 `ServiceLoader.load()` 方法。

SPI 机制应用较为广泛，包括：

- 数据库驱动 JDBC DriveManager。
- 日志库门面 Common-Logging。
- 插件体系。
- Spring 中使用 SPI。

### JDK SPI 在 JDBC 中的应用

JDK 中只定义了一个 java.sql.Driver 接口，具体的实现是由不同数据库厂商来提供的。

以 MySQL 提供的 JDBC 实现包为例：

- 在 mysql-connector-java-*.jar 包中的 META-INF/services 目录下，有一个 java.sql.Driver 文件中只有一行内容，如下所示：

  ```java
  com.mysql.cj.jdbc.Driver
  ```

- 使用 mysql-connector-java-*.jar 包连接 MySQL 数据库的时候，用到如下语句创建数据库连接：

  ```java
  String url = "jdbc:xxx://xxx:xxx/xxx"; 
  Connection conn = DriverManager.getConnection(url, username, pwd); 
  ```

  DriverManager 是 JDK 提供的数据库驱动管理器，其中的代码片段，如下所示：

  ```java
  static { 
      loadInitialDrivers(); 
      println("JDBC DriverManager initialized"); 
  }
  
  private static void loadInitialDrivers() { 
      String drivers = System.getProperty("jdbc.drivers") 
  
      // 使用 JDK SPI 机制加载所有 java.sql.Driver实现类 
      ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class); 
      Iterator<Driver> driversIterator = loadedDrivers.iterator(); 
      while(driversIterator.hasNext()) { 
          driversIterator.next(); 
      } 
  
      String[] driversList = drivers.split(":"); 
      for (String aDriver : driversList) { // 初始化Driver实现类 
          Class.forName(aDriver, true, ClassLoader.getSystemClassLoader()); 
      } 
  } 
  ```

  - 在调用 `getConnection()` 方法的时候，DriverManager 类会被 Java 虚拟机类加载，触发 static 代码块的执行（初始化）。

  - 在 `loadInitialDrivers()` 方法中通过 JDK SPI 扫描 Classpath 下 java.sql.Driver 接口实现类并实例化。

- **MySQL 提供的 com.mysql.cj.jdbc.Driver 实现类**中，同样有一段 static 静态代码块，这段代码会创建一个 com.mysql.cj.jdbc.Driver 对象并注册到 DriverManager.registeredDrivers 集合中。

  ```java
  static { 
      java.sql.DriverManager.registerDriver(new Driver()); 
  }
  ```

  - 在 `getConnection()` 方法中，DriverManager 从该 registeredDrivers 集合中获取对应的 Driver 对象创建 Connection。

    ```java
    private static Connection getConnection(String url, java.util.Properties info, Class<?> caller) throws SQLException { 
        // 省略 try/catch代码块以及权限处理逻辑 
        for(DriverInfo aDriver : registeredDrivers) { 
            Connection con = aDriver.driver.connect(url, info); 
            return con; 
        } 
    } 
    ```



## 对象内存布局

在 Hotspot 虚拟机中，对象在内存中的布局可以分为 3 块区域：

- 对象头（Header）

  - Mark Word（标记字段）

    存储对象的HashCode，分代年龄、锁标志位（锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳）等信息。

    64位 JVM 的 Mark Word 组成如下：

    ![image-20240510013954568](https://gitee.com/fushengshi/image/raw/master/image-20240510013954568.png)

  - Klass Point（类型指针）

    对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

- 实例数据部分（Instance Data）

  实例数据部分是对象真正存储的有效信息，也是在程序中所定义的各种类型的字段内容。

- 对齐填充（Padding）

  对齐填充部分不是必然存在的，也没有什么特别的含义，仅仅起占位作用。

  因为 Hotspot 虚拟机的自动内存管理系统要求对象起始地址必须是 8 字节的整数倍，换句话说就是对象的大小必须是 8 字节的整数倍。



## JVM参数

- `-Xms` / `-XX:InitialHeapSize`

  初始堆大小内存，默认为物理内存的 1/64。

- `-Xmx` / `-XX:MaxHeapSize`

  最大堆分配内存，默认为物理内存的 1/4。

  建议为可用物理内存的 60~75%。

- `-Xmn`

  设置年轻代的大小。默认新生代空间是堆空间的 35%，应该至少为堆空间的 10%。

- `-XX:MetaspaceSize`

  设置元空间大小。

- `-XX:+PrintGCDetails`

- `-XX:SurvivorRatio`

  设置新生代 eden 和 s0/s1 空间的比例，默认 -XX:SurvivorRatio=8，Eden:S0:S1=8:1:1。

- `-XX:NewRatio`

  配置年轻代与老年代在堆结构的占比，默认 -XX:NewRatio=2，新生代占 1，老年代 2，年轻代占整个堆的 1/3。

- `-XX:MaxTenuringThresold`

  配置垃圾最大年龄。

- `-Xss`

  设置单个线程栈的大小。



## JDK命令行工具

- jps（JVM Process Status）

  用于查看所有 Java 进程的启动类、传入参数和 Java 虚拟机参数等信息。

- jstack（Stack Trace for Java）

  生成虚拟机当前时刻的**线程快照**（线程dump），线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合。

  > 线程快照用于定位线程长时间停顿的原因。

  ```shell
  # 打印堆栈
  jstack -l pid
  kill -3
  ```

- jmap（Memory Map for Java）

  > 用于生成堆转储快照。

  ```shell
  jmap -heap {pid}
  jmap -histo {pid} | head -n20  # 查看堆内存中的存活对象，并按空间排序
  
  jmap -dump:format=b,file=/path/dump {pid}
  jmap -dump:live,format=b,file=/path/dump {pid} # 主动触发一次 FullGC 后随即生成 dump 文件
  
  -XX:+HeapDumpOnOutOfMemoryError # 虚拟机在 OOM 异常出现之后自动生成 dump 文件。
  ```

- jstat（JVM Statistics Monitoring Tool）

  用于收集 HotSpot 虚拟机各方面的运行数据，可以显示本地或者远程（需要远程主机提供 RMI 支持）虚拟机进程中的类信息、内存、**垃圾收集**、JIT 编译等运行数据。

  ```shell
  jstat -gc {pid} 1000 1000		# -gc监视java堆状况，每250毫秒查询一次进程2764垃圾收集状态，共查询20次。
  jstat -gcutil {pid} 1000 1000 	# 监视java堆内存使用状况。　
  ```

- jinfo（Configuration Info for Java）

  显示虚拟机配置信息。

  ```shell
  jinfo -flags {pid}
  ```

- jhat（JVM Heap Dump Browser）

  用于分析 heapdump 文件，它会建立一个 HTTP/HTML 服务器，让用户可以在浏览器上查看分析结果。JDK9 移除了 jhat。



## 代码块

基本上代码块分为三种：Static 静态代码块、构造代码块、普通代码块（定义在方法体中）。

```java
public class Demo {
    static {
        System.out.println("静态初始化块");
    }

    {
        System.out.println("初始化代码块");
    }

    public Demo() {
        System.out.println("构造器");
    }
    
    public void method() {
        {
            System.out.println("普通代码块");
        }
    }
}
```

代码块执行顺序：静态代码块 -> 构造代码块 -> 构造函数。

继承中代码块执行顺序：父类静态代码块 ->子类静态代码块 -> 父类构造代码块 -> 父类构造器 -> 子类构造代码块  -> 子类构造器。



## Linux命令

### awk

awk 是一种编程语言，用于在 linux/unix 下对文本和数据进行处理。数据可以来自标准输入（stdin）、一个或多个文件，或其它命令的输出。它支持用户自定义函数和动态正则表达式等先进功能。

```shell
awk 'BEGIN{ commands } pattern{ commands } END{ commands }'
```

- 获取日志中符合过滤条件 partten 的条数

  ```shell
  grep 'pattern' xxx.log | awk 'END{print NR}'
  ```
