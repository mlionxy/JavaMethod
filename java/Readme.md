## Java 中垃圾回收机制中如何判断对象需要回收？常见的 GC 回收算法有哪些？

### 1.Java 中垃圾回收机制中如何判断对象需要回收

**1. 引用计数法**  

引用计数法是通过在对象头中分配一个空间来保存该对象被引用的次数，如果该对象被其他对象引用，则它的引用计数加1，如果删除对该对象的引用，
则它的引用减1，当该对象的引用计数为0时，那么该对象就会被回收。
```
String m = new String("jack");
```
先创建一个字符串，这时候"jack"有一个引用，就是m
```
m = null;
```
然后m设置为null，这时候"jack"的引用计数就等于0，在引用计数算法中，意味着这块内容即将被回收
引用计数算法是将垃圾回收分摊到整个应用程序当中了，而不是在进行垃圾收集时，要挂起整个应用的运行，直到对堆中所有对象的处理都结束。
但是引用计数算法不属于JVM的"Stop-The-World"，所以Java放弃了使用引用计数算法
如果定义两个对象并相互引用，最后置空各自的声明引用，虽然两个对象已不能再访问，但由于它们相互引用，导致他们的引用计数不为0，那么GC收集器永远不会回收它们

**2.可达性分析算法**

可达性分析算法的基本思路是，通过一些被称为引用链(GC Roots)的对象作为起点，从这些节点开始向下搜索，当一个对象到GC Roots没有任何引用链相连时(即GC Roots节点到该节点不可达时)，则证明该对象不可用。
被判定为不可达的对象不一定就会成为可回收对象。被判定为不可达的对象要成为可回收对象必须至少经历两次标记过程，如果在这两次标记过程仍然没有逃脱成为可回收对象的可能性，则会被判定为可回收对象。

在Java语言中，可作为GC Roots的对象包含:
* 1.虚拟机栈中引用的对象。在程序中创建一个对象，对象会在堆中开辟一块空间，同时会将地址作为引用存入到栈中，如果对象生命周期结束了，那么引用就会从栈中出栈。因此虚拟机栈中有引用，说明该对象不可回收。
* 2.方法区中静态属性引用的对象。由于虚拟机栈是线程私有的，因此全局的静态对象static的引用会存入共有的方法区中，将方法区中静态引用作为GC Roots是必须的
* 3.方法区中常量引用的对象。由于该引用创建之后不会修改static final，所以方法区常量池的引用对象也应该作为GC Roots
* 4.本地方法中(Native方法)引用的对象。在使用JNI技术时，调用C与C++的代码会使用Native方法，JVM内存中会有一块本地方法栈保存这些对象的引用，所以本地方法栈中引用的对象也会被作为GC Roots

**3.finallze()方法最终判定对象是否存活**

在可达性分析算法中不可达的对象要经历标记过程判定是否回收
1.第一次标记进行一次筛选
当对象没有覆盖finallze方法，或者finallze方法已经被虚拟机调用过，虚拟机会将这两种情况视为没有必要执行。对象会被回收

2.第二次标记
如果对象被判定为需要执行finallze方法时，对象会被放在一个名为F-Queue队列中，稍后会由一个Finallze线程去执行
如果对象在Finallze中重新与GC Roots上的任何一个对象建立关联，那么会将该对象移除即将回收的集合，如果未建立关联，则被回收

### 2.常见的GC回收算法

**1.标记清除算法**

标记清除算法分为两个阶段: 标记阶段和清除阶段，标记阶段是标记出所有需要被回收的对象，清除阶段就是回收标记的对象占用的空间
标记清除算法比较容易实现，但容易产生内存碎片，碎片太多会导致下一次大对象分配空间无法找到足够的空间从而触发新的垃圾回收

**2.复制算法**

复制算法将可用内存按容量划分为大小相等的两块，每次只使用其中一块，当一块内存使用完将存活的对象复制到另一块上，再将已使用的内存空间一次清理掉。

**3.标记整理算法**

标记阶段标记完成后，将存活的对象都向一端移动，然后清理掉边界以外的内存。

**4.分代收集算法**

分代收集算法是目前大部分JVM的垃圾收集器采用的算法。它的核心思想是根据存活的生命周期将内存划分为若干个不同区域。
一般情况下将堆区划分为老年代和新生代，老年代的特点是每次垃圾回收时只有少量的对象被回收，而新生代的特点是每次垃圾回收有大量的对象被回收。
目前大部分垃圾收集器对新生代采用复制算法，因为新生代大部分对象要回收，需要复制的次数较少，一般会将新生代划分为一块较大的Eden和两块较小的Survivor,
每次使用Eden和一块Survivor空间，当进行回收时，将Eden和Survivor中还存活的对象复制到另一块Survivor空间中，然后清理掉Eden和刚才使用过的Survivor空间。
由于老年代的特点是每次回收都只回收少量对象，一般使用的是标记整理算法

在堆区之外还有一个代就是永久代，对永久代的回收主要回收两部分内容：废弃常量和无用的类。

## hashmap 和 hashtable 的区别是什么？

**1.继承父类不同**

HashMap是继承自AbstractMap类，而HashTable是继承自Dictionary类。不过它们都实现了同时实现了map、Cloneable（可复制）、Serializable（可序列化）这三个接口

**2.对外提供接口不同**

Hashtable比HashMap多提供了elments() 和contains() 两个方法。

**3.对Null key 和Null value的支持不同**

Hashtable既不支持Null key也不支持Null value。
HashMap中，null可以作为键，这样的键只有一个；可以有一个或多个键所对应的值为null。当get()方法返回null值时，可能是 HashMap中没有该键，也可能使该键所对应的值为null。因此，在HashMap中不能由get()方法来判断HashMap中是否存在某个键， 而应该用containsKey()方法来判断。

**4.线程安全性不同**

Hashtable是线程安全的，它的每个方法中都加入了Synchronize方法。在多线程并发的环境下，可以直接使用Hashtable，不需要自己为它的方法实现同步
HashMap不是线程安全的，在多线程并发的环境下，可能会产生死锁等问题。

**5.遍历方式的内部实现上不同**

Hashtable、HashMap都使用了 Iterator。Hashtable还使用了Enumeration的方式 。

**6.初始容量大小和每次扩充容量大小的不同**

Hashtable默认的初始大小为11，之后每次扩充，容量变为原来的2n+1。HashMap默认的初始化大小为16。之后每次扩充，容量变为原来的2倍。

**7.计算hash值的方法不同**

Hashtable直接使用对象的hashCode。hashCode是JDK根据对象的地址或者字符串或者数字算出来的int类型的数值。然后再使用除留余数发来获得最终的位置。Hashtable在计算元素的位置时需要进行一次除法运算，而除法运算是比较耗时的。
HashMap为了提高计算效率，将哈希表的大小固定为了2的幂，这样在取模预算时，不需要做除法，只需要做位运算。位运算比除法的效率要高很多。

**8.解决hash冲突方式不同**

在HashMap中如果冲突数量小于8，则是以链表方式解决冲突。而当冲突大于等于8时，就会将冲突的Entry转换为红黑树进行存储。而又当数量小于6时，则又转化为链表存储。
在HashTable中，都是以链表方式存储。

## HashMap 与 ConcurrentHashMap 的实现原理是怎样的？ConcurrentHashMap 是如何保证线程安全的？

### HashMap实现原理

**1.存储结构**

HashMap采用Entry数组存储key-value对，每一个键值对组成了一个Entry实体，Entry类实际上是一个单向的链表结构，它具有next指针，可以链接下一个Entry实体，以此来解决hash冲突的问题。
数组存储区间是连续的，占用内存严重，故空间复杂度很大。但数组的二分查找时间复杂度小，为O(1),数组的特点是，寻址容易，插入删除困难。
链表存储区间离散，占用内存比较宽松，故空间复杂度小，但时间复杂度很大，达O(N)。链表的特点是，寻址困难，插入和删除容易。
HashMap数据结构是由数组+链表组成，一个长度16的数组中，每一个元素存储的是链表的头节点，通过计算hash(key.hashCode())%len获得，也就是元素key的hash值对数组长度取余得到。比如12，28，108，140它们计算结果都是12所以它们存储在数组下标为12的位置。
Entry里面主要属性key，value，hash，next，第一个键值对A进来，通过计算其key的hash得到的index=0，Entry[0] = A。键值对B，通过计算其index等于0，HashMap会这样做:B.next = A,Entry[0] = B,如果又进来C,index也等于0,那么C.next = B,Entry[0] = C

**2.JDK 1.8的 改变**

在Jdk1.8中HashMap数据结构存储方式为数组+链表+红黑树的存储方式，当链表的长度超过8时，将链表转换为红黑树。

### ConcurrentHashMap实现原理，如何保证线程安全

**1.存储结构**

ConcurrentHashMap和HashMap实现上类似，最主要的差别是ConcurrentHashMap采用了分段锁（Segment），每个分段锁维护着几个桶（HashEntry），
多个线程可以同时访问不同分段锁上的桶，从而使其并发度更高（并发度就是 Segment 的个数）。Segment继承自ReentrantLock。默认的并发级别为16，也就是说默认创建16个Segment。

**2.size操作**

每个Segment维护了一个count变量来统计该Segment中的键值对个数。在执行size操作时，需要遍历所有Segment然后把count累计起来。ConcurrentHashMap在执行size操作时先尝试不加锁，如果连续两次不加锁操作得到的结果一致，那么可以认为这个结果是正确的。
尝试次数使用RETRIES_BEFORE_LOCK定义，该值为2，retries初始值为-1，因此尝试次数为3。如果尝试的次数超过3次，就需要对每个Segment加锁。

**3.JDK 1.8 的改动**

JDK1.7使用分段锁机制来实现并发更新操作，核心类为Segment，它继承自重入锁ReentrantLock，并发度与Segment数量相等。JDK1.8使用了CAS操作来支持更高的并发度，在CAS操作失败时使用内置锁synchronized。
JDK1.8的实现也在链表过长时会转换为红黑树。

## 简述 Java 的反射机制及其应用场景

### 反射的机制
Java反射机制的核心是在程序运行时动态加载类并获取类的详细信息，从而操作类或对象的属性和方法。本质是JVM得到class对象之后，再通过class对象进行反编译，从而获取对象的各种信息。
Class和java.lang.reflect一起对反射提供了支持，java.lang.reflect类库主要包含了以下三个类：
* Field：可以使用get()和set()方法读取和修改Field对象关联的字段。
* Method：可以使用invoke()方法调用与Method对象关联的方法。
* Constructor：可以用Constructor的newInstance()创建新的对象。

### 反射的应用

* 反射让开发人员可以通过外部类的全路径名来创建对象，并使用这些类，实现一些扩展的功能。
* 反射让开发人员可以枚举出类的全部成员，包括构造函数，属性，方法。以帮助开发者写出正确的代码。
* 测试时可以利用反射API访问类的私有成员。以保证测试代码覆盖率。

## Java 类的加载流程是怎样的？什么是双亲委派机制？

### Java 类的加载流程

JVM把class文件加载在内存中，并对数据进行校验，准备，解析，初始化，最终形成JVM可以直接使用的Java类型的过程。
加载-验证-准备-解析-初始化-使用-卸载

**1.加载**

把class字节码文件加载到内存中，并将这些数据转换成方法区中的运行时数据（静态变量，静态代码块，常量池等），在堆中生成一个Class类对象代表这个类，作为方法区类数据的访问入口。

**2.链接**

将Java类的二进制代码合并到JVM的运行状态之中。

* 验证

确保加载的类信息符合JVM规范，没有安全方面问题。

* 准备

正式为类变量(static变量)分配内存并设置类变量初始值的阶段，这些内存都将在方法区中进行分配。注意此时的设置初始值为默认值，具体赋值在初始化阶段完成。

* 解析

虚拟机常量池内的符号引用替换为直接引用（地址引用）的过程。

**3.初始化**

初始化阶段是执行类构造器方法的过程。类构造器方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块(static块)中的语句合并产生的。
当初始化一个类的时候，如果发现其父类还没有进行过初始化、则需要先初始化其父类。虚拟机会保证一个类的方法在多线程环境中被正确加锁和同步

### 双亲委派机制

某个特定的类加载器接收到类加载的请求时，会将加载任务委托给自己的父类，直到最高级父类引导类加载器，如果父类能够加载就加载，不能加载则返回到子类进行加载。如果都不能加载则报错。ClassNotFoundException。
双亲委派机制是为了保证Java核心库的类型安全。这种机制保证不会出现用户自己能定义java.lang.Object类等的情况。例如，用户定义了java.lang.String，那么加载这个类时最高级父类会首先加载，发现核心类中也有这个类，那么就加载了核心类库，而自定义的永远都不会加载。

## HashMap 实现原理，为什么使用红黑树？

**HHashMap 实现原理**

HashMap采用Entry数组存储key-value对，每一个键值对组成了一个Entry实体，Entry类实际上是一个单向的链表结构，它具有next指针，可以链接下一个Entry实体，以此来解决hash冲突的问题。
数组存储区间是连续的，占用内存严重，故空间复杂度很大。但数组的二分查找时间复杂度小，为O(1),数组的特点是，寻址容易，插入删除困难。
链表存储区间离散，占用内存比较宽松，故空间复杂度小，但时间复杂度很大，达O(N)。链表的特点是，寻址困难，插入和删除容易。
HashMap数据结构是由数组+链表组成，一个长度16的数组中，每一个元素存储的是链表的头节点，通过计算hash(key.hashCode())%len获得，也就是元素key的hash值对数组长度取余得到。比如12，28，108，140它们计算结果都是12所以它们存储在数组下标为12的位置。
Entry里面主要属性key，value，hash，next，第一个键值对A进来，通过计算其key的hash得到的index=0，Entry[0] = A。键值对B，通过计算其index等于0，HashMap会这样做:B.next = A,Entry[0] = B,如果又进来C,index也等于0,那么C.next = B,Entry[0] = C

**为什么使用红黑树**

在JDK1.8之前HashMap使用数组+链表实现。使用链表处理冲突，同一hash值的数据都存在一个链表里。当一个桶中的元素较多，即hash值相等的元素较多时，通过key值依次查找的效率较低。
在JDK1.8之后HashMap使用数组+链表+红黑树实现。当链表的长度超过阈值8时，将链表转换为红黑树，以加快检索速度。
受随机分布的hashCode影响，链表中的节点遵循泊松分布，根据统计，链表中节点数是8的概率较低，此时链表的性能较差。所以在这种情况下，会把链表转变为红黑树。因为链表转换为红黑树是需要消耗性能的，为了挽回性能，权衡之下，才使用红黑树，提高性能。在大部分情况下，hashmap还是使用的链表，如果是理想的均匀分布，节点数不到8，hashmap就自动扩容

## Synchronized 关键字底层是如何实现的？它与 Lock 相比优缺点分别是什么？

### Synchronized 底层原理

synchronized修饰的方法在字节码中添加了一个ACC_SYNCHRONIZED的flags,
同步代码块则是在同步代码块前插入monitorenter，在同步代码块结束后插入monitorexit。
这两者的处理是分别是这样的：当线程执行到某个方法时，JVM会去检查该方法的ACC_SYNCHRONIZED访问标志是否被设置，
如果设置了那线程会去获取这个对象所对应的monitor对象（每一个对象都有且仅有一个与之对应的monitor对象）,获取成功后才执行方法体，
方法执行完再释放monitor对象，在这一期间，任何其他线程都无法获得这个monitor对象。
而线程执行同步代码块时遇到的monitorenter和monitorexit指令依赖monitor对象完成。
这两者实现的方式本质上无区别，只是方法的同步是一种隐式的方式，不通过字节码实现。
同步和monitor有关，而monitor则和对象头有关。

**synchronized影响性能的原因**

* 1、加锁解锁操作需要额外操作；
* 2、互斥同步对性能最大的影响是阻塞的实现，因为阻塞涉及到的挂起线程和恢复线程的操作都需要转入内核态中完成（用户态与内核态的切换的性能代价是比较大的）

### Synchronized 与 Lock 区别

* synchronized是Java语法的一个关键字，加锁的过程是在JVM底层进行。Lock是一个类，是JDK应用层面的，在JUC包里有丰富的API。
* Lock锁适合大量同步的代码的同步问题，synchronized锁适合代码少量的同步问题。
* Lock锁有丰富的API能知道线程是否获取锁成功，而synchronized不能。
* synchronized能修饰方法和代码块，Lock锁只能锁住代码块。
* Lock锁有丰富的API，可根据不同的场景，在使用上更加灵活。
* synchronized是非公平锁，而Lock锁既有非公平锁也有公平锁，可以由开发者通过参数控制。
* synchronized会自动释放锁(a 线程执行完同步代码会释放锁 ；b 线程执行过程中发生异常会释放锁)，Lock需在finally中手工释放锁（unlock()方法释放锁），否则容易造成线程死锁；

## HashMap 1.7 / 1.8 的实现区别

* 1.new HashMap():底层没有创建一个长度为16的数组
* 2.jdk 8底层的数组是：Node[],而非Entry[]
* 3.首次调用put()方法时，底层创建长度为16的数组
* 4.jdk7底层结构只有：数组+链表。jdk8中底层结构：数组+链表+红黑树。
	形成链表时，七上八下（jdk7:新的元素指向旧的元素。jdk8：旧的元素指向新的元素）
	当数组的某一个索引位置上的元素以链表形式存在的数据个数 > 8 且当前数组的长度 > 64时，此时此索引位置上的所数据改为使用红黑树存储。
	红黑树的平均查找长度是log(n)，长度为8，查找长度为log(8)=3，链表的平均查找长度为n/2，当长度为8时，平均查找长度为8/2=4，这才有转换成树的必要；链表长度如果是小于等于6，6/2=3，虽然速度也很快的，但是转化为树结构和生成树的时间并不会太短。

## 简述 BIO, NIO, AIO 的区别

### BIO 

BIO是JDK1.4之前的传统IO模型，本身是同步阻塞模式。线程发起IO请求后，一直阻塞IO，直到缓冲区数据就绪后，再进入下一步操作。针对网络通信都是一请求一答应的方式，
虽然简化了上层的应用开发，但在性能和可靠性方面存在着巨大瓶颈，试想一下如果每个请求都需要新建一个线程来专门处理，那么在高并发的情况下，机器资源很快就会被耗尽

### NIO

NIO是同步非阻塞的IO模型。线程发起io请求后，立即返回（非阻塞io）。同步指的是必须等待IO缓冲区的数据准备就绪，而非阻塞指的是，用户线程不原地等待缓冲区，可以先做
一些其他操作，但是要定时轮询检查IO缓冲区的数据是否就绪。Java中的NIO是new IO的意思。其实是NIO加上IO多路复用技术。普通的NIO是线程轮询查看一个IO缓冲区是否就绪，而
Java中的new IO指的是线程轮询的去查看一堆IO缓冲区中哪些就绪，只是一种IO多路复用的思想。IO多路复用模型中，将检查IO数据是否就绪的任务，交给系统级别的select或epoll模型，由系统进行监控，减轻用户线程负担。

NIO主要有buffer、channel、selector三种技术的整合，通过零拷贝的buffer取得数据，每一个客户端通过channel在selector上进行注册。服务端不断轮询channel来获取客户端的信息。channel上有connect，accept，read
，write四种状态标识。根据标识来进行后续操作。

### AIO

AIO是真正意义上的异步非阻塞IO模型。上述NIO实现中，需要用户线程定时轮询，去检查IO缓冲区数据是否就绪，占用应用线程资源，其实轮询相当于还是阻塞的，并非真正解放当前线程，还需要
去查询哪些IO准备就绪。而真正的理想的异步非阻塞IO应该让内核系统完成，用户线程只需要告诉内核，当缓冲区就绪后，通知我或者执行我交给你的回调函数。

## Java 中接口和抽象类的区别

* 接口（interface）和抽象类（abstract class）是支持抽象类定义的两种机制。
* 接口是公开的，不能有私有的方法或变量，接口中所有的方法都没有方法体，通过关键字interface实现
* 抽象类是可以有私有方法或私有变量的，通过把类或者类中的方法声明为abstract来表示一个类是抽象类，被声明为抽象的方法不能包含方法体。子类实现方法必须含有相同的或者更低的访问级别。抽象类的子类为父类中所有抽象方法的具体实现。
* 接口可以被看作是抽象类的变体，接口中所有的方法都是抽象的，可以通过接口来间接的实现多重继承。接口中的成员变量都是static final类型，由于抽象类可以包含部分方法的实现，所以，在一些场合下抽象类比接口更有优势。

### 相同点

* 1.都不能被实例化
* 2.接口的实现类或抽象类的子类都只有实现了接口或抽象类的方法后才能实例化

### 不同点

* 1.接口只有定义，不能有方法的实现，java1.8中可以定义default方法体，而抽象类可以有定义与实现，方法可在抽象类中实现。
* 2.实现接口的关键字为implements，继承抽象类的关键字为extends。一个类可以实现多个接口，但一个类只能继承一个抽象类。所以，使用接口可以间接的实现多重继承。
* 3.接口强调特定功能的实现，而抽象类强调所属关系。
* 4.接口成员变量默认为public static final，必须赋初始值，不能被修改，其所有的成员方法都是public，abstract。抽象类中成员变量默认为default，可在子类中被重新定义，也可被重新赋值，抽象方法被abstract修饰，不能被private、static、synchronized和native等修饰，必须以分号结尾，不带花括号。
* 5.接口被用于常用的功能，便于日后维护和添加删除，而抽象类更倾向于充当公共类的角色，不适用于日后重新对立面的代码修改。功能需要累积时用抽象类，不需要累积时用接口。

## Java 常见锁有哪些？ReetrantLock 是怎么实现的？

### Java 常见锁有哪些

**Synchronized和Lock**

* Synchronized是一个：非公平，悲观，独享，互斥，可重入的重量级锁
* 以下两个锁都在JUC包下，是API层面上的实现
* ReentrantLock，它是一个：默认非公平但可实现公平的，悲观，独享，互斥，可重入，重量级锁。
* ReentrantReadWriteLocK，它是一个，默认非公平但可实现公平的，悲观，写独享，读共享，读写，可重入，重量级锁。

**悲观锁&乐观锁**

乐观锁与悲观锁不是指具体的什么类型的锁，而是指看待并发同步的角度。悲观锁认为对于同一个数据的并发操作，一定是会发生修改的，哪怕没有修改，也会认为修改。因此对于同一个数据的并发操作，悲观锁采取加锁的形式。悲观的认为，不加锁的并发操作一定会出问题。
乐观锁则认为对于同一个数据的并发操作，是不会发生修改的。在更新数据的时候，会采用尝试更新，不断重新的方式更新数据。乐观的认为，不加锁的并发操作是没有事情的。从上面的描述我们可以看出，悲观锁适合写操作非常多的场景，乐观锁适合读操作非常多的场景，不加锁会带来大量的性能提升。
悲观锁在Java中的使用，就是利用各种锁。乐观锁在Java中的使用，是无锁编程，常常采用的是CAS算法，典型的例子就是原子类，通过CAS自旋实现原子操作的更新。

* 悲观锁适合写操作多的场景，先加锁可以保证写操作时数据正确。
* 乐观锁适合读操作多的场景，不加锁的特点能够使其读操作的性能大幅提升。

**公平锁/非公平锁**

公平锁是指多个线程按照申请锁的顺序来获取锁。非公平锁是指多个线程获取锁的顺序并不是按照申请锁的顺序，有可能后申请的线程比先申请的线程优先获取锁。有可能，会造成优先级反转或者饥饿现象。对于Java ReentrantLock而言，通过构造函数指定该锁是否是公平锁，
默认是非公平锁。非公平锁的优点在于吞吐量比公平锁大。对于Synchronized而言，也是一种非公平锁。由于其并不像ReentrantLock是通过AQS的来实现线程调度，所以并没有任何办法使其变成公平锁。

**独享锁/共享锁**

独享锁是指该锁一次只能被一个线程所持有。共享锁是指该锁可被多个线程所持有。对于Java ReentrantLock而言，其是独享锁。但是对于Lock的另一个实现类ReentrantReadWriteLock，其读锁是共享锁，其写锁是独享锁。
读锁的共享锁可保证并发读是非常高效的，读写，写读 ，写写的过程是互斥的。独享锁与共享锁也是通过AQS来实现的，通过实现不同的方法，来实现独享或者共享。对于Synchronized而言，当然是独享锁。

**可重入锁**

可重入锁又名递归锁，是指在同一个线程在外层方法获取锁的时候，在进入内层方法会自动获取锁。说的有点抽象，下面会有一个代码的示例。对于Java ReentrantLock而言, 他的名字就可以看出是一个可重入锁，其名字是Reentrant Lock重新进入锁。对于Synchronized而言,也是一个可重入锁。可重入锁的一个好处是可一定程度避免死锁。

### ReetrantLock 实现原理

ReentrantLock，重入锁，是JDK5中添加在并发包下的一个高性能的工具。ReentrantLock支持同一个线程在未释放锁的情况下重复获取锁。

ReentrantLock的实现基于队列同步器（AQS），ReentrantLock的可重入功能基于AQS的同步状态：state。
当某一线程获取锁后，将state值+1，并记录下当前持有锁的线程，再有线程来获取锁时，判断这个线程与持有锁的线程是否是同一个线程，如果是，将state值再+1，如果不是，阻塞线程。 当线程释放锁时，将state值-1，当state值减为0时，表示当前线程彻底释放了锁，然后将记录当前持有锁的线程的那个字段设置为null，并唤醒其他线程，使其重新竞争锁。

## Java 线程间有多少通信方式？

### 1.volatile关键字方式

volatile有两大特性，一是可见性，二是有序性，禁止指令重排序，其中可见性就是可以让线程之间进行通信。
volatile语义保证线程可见性有两个原则保证
* 所有volatile修饰的变量一旦被某个线程更改，必须立即刷新到主内存
* 所有volatile修饰的变量在使用之前必须重新读取主内存的值

### 2.等待/通知机制

等待通知机制是基于wait和notify方法来实现的，在一个线程内调用该线程锁对象的wait方法，线程将进入等待队列进行等待直到被通知或者被唤醒。

### 3.join方式

join其实合理理解成是线程合并，当在一个线程调用另一个线程的join方法时，当前线程阻塞等待被调用join方法的线程执行完毕才能继续执行，所以join的好处能够保证线程的执行顺序，但是如果调用线程的join方法其实已经失去了并行的意义，虽然存在多个线程，但是本质上还是串行的，最后join的实现其实是基于等待通知机制的。

### 4.threadLocal方式

threadLocal方式的线程通信，不像以上三种方式是多个线程之间的通信，它更像是一个线程内部的通信，将当前线程和一个map绑定，在当前线程内可以任意存取数据，减省了方法调用间参数的传递。

## 简述 ArrayList 与 LinkedList 的底层实现以及常见操作的时间复杂度

### ArrayList实现原理 

ArrayList是通过数组实现，一旦我们实例化ArrayList无参数构造函数默认为数组初始化长度为10，add方法底层实现如果增加的元素个数超过了10个，那么ArrayList底层会新生一个数组，长度为原数组的1.5倍+1，然后将原数组的
内容复制到新数组当中，并且后续增加的内容都会放到新数组当中。当新数组无法容纳增加的元素时，重复该过程。一旦数组超出长度，就开始扩容数组。扩容数组调用的方法 Arrays.copyOf(objArr, objArr.length + 1)

**时间复杂度**

ArrayList 是线性表（数组）
* get() 直接读取第几个下标，复杂度 O(1)
* add(E) 添加元素，直接在后面添加，复杂度O（1）
* add(index, E) 添加元素，在第几个元素后面插入，后面的元素需要向后移动，复杂度O（n）
* remove（）删除元素，后面的元素需要逐个移动，复杂度O（n）

### LinkedList实现原理

LinkedList是以双向链表实现，链表无容量限制（但是双向链表本身需要消耗额外的链表指针空间来操作），其内部主要成员为first和last两个Node节点，在每次修改列表时用来指引当前双向链表的首尾部位，
所以LinkedList不仅仅实现了List接口，还实现了Deque双端队列接口（该接口是Queue队列的子接口），故LinkedList自动具备双端队列的特性，当我们使用下标方式调用列表的get(index)、set(index,e)方法时需要遍历链表将指针移动到位进行访问（会判断index是否大于链表长度的一半决定是首部遍历还是尾部遍历，访问的复杂度为O(N/2)）

**时间复杂度**

LinkedList 是链表的操作
* get() 获取第几个元素，依次遍历，复杂度O(n)
* add(E) 添加到末尾，复杂度O(1)
* add(index, E) 添加第几个元素后，需要先查找到第几个元素，直接指针指向操作，复杂度O(n)
* remove（）删除元素，直接指针指向操作，复杂度O(1)

## 简述 JVM 的内存模型 JVM 内存是如何对应到操作系统内存的？

### JVM内存模型主要分为私有和共享两种：
* 线程私有：程序计数器、虚拟机栈、本地方法栈
* 线程共享：堆、方法区

1. 程序计数器：当前线程执行的字节码的行号指示器，用于选取下一条要执行的指令。
2. 虚拟机栈：存储了局部变量表，操作数栈，动态链接，方法出口等信息。
3. 本地方法栈：和虚拟机栈所发挥的作用非常相似，区别是：虚拟机栈为虚拟机执行Java方法服务，而本地方法栈则为虚拟机使用到的Native方法服务。 
4. 堆：存放对象实例，涉及到内存的分配(new关键字，反射等)与回收(回收算法，收集器等)，几乎所有的对象都是在堆中分配。
5. 方法区：用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。


###  JVM 内存是如何对应到操作系统内存的

jvm的内存是分布在操作系统的堆中，jvm的设计的模型其实就是操作系统的模型，对于操作系统来说，JVM就是一个程序；对于class文件来说，jvm就是一个操作系统，jvm的方法区，也就相当于操作系统的硬盘区；

## 说说 List,Set,Map 三者的区别及如何选用？

* list:存储的元素是有序、可重复的。
* set:存储的元素是无序、不可重复的。
* Map:存储的元素是Key-value形式的，key无序且不可重复，value无序可重复

### list:
* ArrayList：底层使用 Object[] 存储,线程不安全
* Vector：底层使用 Object[] 存储，线程安全
* LinkedList：底层使用 双向链表 存储，线程不安全

### set:
* HashSet：底层是 HashMap，线程不安全的
* LinkedHashSet：HashSet 的子类，线程不安全的
* TreeSet：底层是红黑树，线程不安全的

### Map:
* HashMap：底层是 数组+红黑树/链表 ，线程不安全
* LinkedHashMap：HashMap的子类，线程不安全
* HashTable 底层是 数组+链表 ，线程安全的
* TreeMap：线程不安全的

### 如何选用集合

主要根据集合的特点来选用，比如我们需要根据键值获取到元素值时就选用 Map 接口下的集合，需要排序时选择 TreeMap,不需要排序时就选择 HashMap,需要保证线程安全就选用 ConcurrentHashMap。
当我们只需要存放元素值时，就选择实现Collection 接口的集合，需要保证元素唯一时选择实现 Set 接口的集合比如 TreeSet 或 HashSet，不需要就选择实现 List 接口的比如 ArrayList 或 LinkedList，然后再根据实现这些接口的集合的特点来选用。

## 手写生产者消费者模型

五种方式实现生产者消费者模型

### 1.wait()和notify()方法

- 使用synchronized来进行同步；
- 缓冲区满和为空时都调用wait()方法等待；
- 当生产者生产了一个产品或者消费者消费了一个产品之后会唤醒所有线程。

```java
package BasicKnowleage;

/** Method 1: synchronized + wait & notifyAll **/
public class ProducerConsumer01 {
  private static Integer count = 0;
  private static String LOCK = "lock";
  private static final Integer FULL = 10;
  private static final Integer EMPTY = 0;

  class Producer implements Runnable {
    @Override
    public void run() {
      for(int i = 0; i < 10; i++) {
        try {
          Thread.sleep(3000);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        synchronized (LOCK) {
          while (count == FULL) {
            try {
              LOCK.wait();
            } catch (InterruptedException e) {
              e.printStackTrace();
            }
          }
          count ++;
          System.out.println(Thread.currentThread().getName() + " producer number: " + count);
          LOCK.notifyAll();
        }
      }
    }
  }

  class Consumer implements Runnable {

    @Override
    public void run() {
      for (int i = 0; i < 10; i++) {
        try {
          Thread.sleep(3000);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        synchronized (LOCK){
          while (count == EMPTY) {
            try {
              LOCK.wait();
            } catch (InterruptedException e) {
              e.printStackTrace();
            }
          }
          count --;
          System.out.println(Thread.currentThread().getName() + " consumer number: " + count);
          LOCK.notifyAll();
        }
      }
    }
  }

  public static void main(String[] args) {
    ProducerConsumer01 producerConsumer01 = new ProducerConsumer01();
    new Thread(producerConsumer01.new Producer()).start();
    new Thread(producerConsumer01.new Consumer()).start();
    new Thread(producerConsumer01.new Producer()).start();
    new Thread(producerConsumer01.new Consumer()).start();
    new Thread(producerConsumer01.new Producer()).start();
    new Thread(producerConsumer01.new Consumer()).start();
    new Thread(producerConsumer01.new Producer()).start();
    new Thread(producerConsumer01.new Consumer()).start();
  }

}
```

### 2. 可重入锁ReentrantLock
- 创建一个锁对象ReentrantLock，注意；需要释放锁
- 为这个锁创建两个条件Condition变量，一个为缓冲区notFull，一个为缓冲区notEmpty


``` java
package BasicKnowleage;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/** Method 2: ReentrantLock + wait & notifyAll **/
public class ProducerConsumer02 {

  private static Integer count = 0;
  private static ReentrantLock lock = new ReentrantLock();
  private static final Integer FULL = 10;
  private static final Integer EMPTY = 0;
  private static Condition notFull = lock.newCondition();
  private static Condition notEmpty = lock.newCondition();

  class Producer implements Runnable {
    @Override
    public void run() {
      for (int i = 0; i < 10; i++) {
        try {
          Thread.sleep(3000);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        lock.lock();
        try {
          while (count == FULL) {
            try {
              notFull.await();
            } catch (InterruptedException e) {
              e.printStackTrace();
            }
          }
          count ++;
          System.out.println(Thread.currentThread().getName() + " producer number: " + count);
          notEmpty.signal();
        } finally {
          lock.unlock();
        }
      }
    }
  }

  class Consumer implements Runnable {
    @Override
    public void run() {
      for (int i =0; i< 10; i++) {
        try {
          Thread.sleep(3000);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        lock.lock();
        try {
          while (count == EMPTY) {
            try {
              notEmpty.await();
            } catch (InterruptedException e) {
              e.printStackTrace();
            }
          }
          count --;
          System.out.println(Thread.currentThread().getName() + " consumer number: " + count);
          notFull.signal();
        } finally {
          lock.unlock();
        }
      }
    }
  }

  public static void main(String[] args) {
    ProducerConsumer02 producerConsumer02 = new ProducerConsumer02();
    new Thread(producerConsumer02.new Producer()).start();
    new Thread(producerConsumer02.new Consumer()).start();
    new Thread(producerConsumer02.new Producer()).start();
    new Thread(producerConsumer02.new Consumer()).start();
    new Thread(producerConsumer02.new Producer()).start();
    new Thread(producerConsumer02.new Consumer()).start();
    new Thread(producerConsumer02.new Producer()).start();
    new Thread(producerConsumer02.new Consumer()).start();
  }
}
```

### 3. 阻塞队列BlockingQueue
- 定义一个阻塞队列，其中lockingQueue的put和take时阻塞的方法


``` java
package BasicKnowleage;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

/** Method 3:  BlockingQueue **/
public class ProducerConsumer03 {
  private static Integer count = 0;
  final BlockingQueue blockingDeque = new ArrayBlockingQueue<Integer>(10);

  class Producer implements Runnable {
    @Override
    public void run() {
      for (int i = 0; i < 10; i++) {
        try {
          Thread.sleep(3000);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        try {
          blockingDeque.put(1);
          count ++;
          System.out.println(Thread.currentThread().getName() + " producer number: " + count);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    }
  }

  class Consumer implements Runnable {
    @Override
    public void run() {
      for (int i =0; i< 10; i++) {
        try {
          Thread.sleep(3000);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        try {
          blockingDeque.take();
          count --;
          System.out.println(Thread.currentThread().getName() + " consumer number: " + count);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    }
  }

  public static void main(String[] args) {
    ProducerConsumer03 producerConsumer03 = new ProducerConsumer03();
    new Thread(producerConsumer03.new Producer()).start();
    new Thread(producerConsumer03.new Consumer()).start();
    new Thread(producerConsumer03.new Producer()).start();
    new Thread(producerConsumer03.new Consumer()).start();
    new Thread(producerConsumer03.new Producer()).start();
    new Thread(producerConsumer03.new Consumer()).start();
    new Thread(producerConsumer03.new Producer()).start();
    new Thread(producerConsumer03.new Consumer()).start();
  }

}
```

### 4. 信号量Semaphore
- 添加两个信号量notFull和notEmpty作为许可集；
- 可以使用acquire()方法获得一个许可，当许可不足时会被阻塞，release()添加一个许可；
- 加入了另外一个mutex信号量，维护生产者消费者之间的同步关系，保证生产者和消费者之间的交替进行


``` java
package BasicKnowleage;

import java.util.concurrent.Semaphore;

/** Method 4:  Semaphore **/
public class ProducerConsumer04 {
  private static Integer count = 0;
  final Semaphore notFull = new Semaphore(10);
  final Semaphore notEmpty = new Semaphore(0);
  final Semaphore mutex = new Semaphore(1);

  class Producer implements Runnable {
    @Override
    public void run() {
      for (int i = 0; i < 10; i++) {
        try {
          Thread.sleep(3000);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        try {
          notFull.acquire();
          mutex.acquire();
          count ++;
          System.out.println(Thread.currentThread().getName() + " producer number: " + count);
        } catch (InterruptedException e) {
          e.printStackTrace();
        } finally {
          notFull.release();
          mutex.release();
        }
      }
    }
  }

  class Consumer implements Runnable {
    @Override
    public void run() {
      for (int i =0; i< 10; i++) {
        try {
          Thread.sleep(3000);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        try {
          notEmpty.acquire();
          mutex.acquire();
          count --;
          System.out.println(Thread.currentThread().getName() + " consumer number: " + count);
        } catch (InterruptedException e) {
          e.printStackTrace();
        } finally {
          notEmpty.release();
          mutex.release();
        }
      }
    }
  }

  public static void main(String[] args) {
    ProducerConsumer04 producerConsumer04 = new ProducerConsumer04();
    new Thread(producerConsumer04.new Producer()).start();
    new Thread(producerConsumer04.new Consumer()).start();
    new Thread(producerConsumer04.new Producer()).start();
    new Thread(producerConsumer04.new Consumer()).start();
    new Thread(producerConsumer04.new Producer()).start();
    new Thread(producerConsumer04.new Consumer()).start();
    new Thread(producerConsumer04.new Producer()).start();
    new Thread(producerConsumer04.new Consumer()).start();
  }
}

```
 
### 5. 管道输入输出流PipedInputStream和PipedOutputStream

- 先创建一个管道输入流和管道输出流，然后将输入流和输出流进行连接
- 生产者线程往管道输出流中写入数据，消费者在管道输入流中读取数据，这样就可以实现了不同线程间的相互通讯
- **注意**：这种方式在生产者和生产者、消费者和消费者之间不能保证同步，也就是说在一个生产者和一个消费者的情况下是可以生产者和消费者之间交替运行的，多个生成者和多个消费者者之间则不行


``` java
package BasicKnowleage;

import java.io.IOException;
import java.io.PipedInputStream;
import java.io.PipedOutputStream;


public class ProducerConsumer05 {
  final PipedInputStream pipedInputStream = new PipedInputStream();
  final PipedOutputStream pipedOutputStream = new PipedOutputStream();
  {
    try {
      pipedInputStream.connect(pipedOutputStream);
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

  class Producer implements Runnable {
    @Override
    public void run() {
        try {
          while(true) {
            Thread.sleep(1000);
            int num = (int) (Math.random() * 255);
            System.out.println(Thread.currentThread().getName() + " producer number: " + num);
            pipedOutputStream.write(num);
            pipedOutputStream.flush();
          }
        } catch (Exception e) {
          e.printStackTrace();
        } finally {
          try {
            pipedOutputStream.close();
            pipedInputStream.close();
          } catch (IOException e) {
            e.printStackTrace();
          }
        }
      }
  }
  class Consumer implements Runnable {
    @Override
    public void run() {
        try {
          Thread.sleep(1000);
          int num = pipedInputStream.read();
          System.out.println(Thread.currentThread().getName() + " consumer number: " + num);
        } catch (Exception e) {
          e.printStackTrace();
        } finally {
          try {
            pipedOutputStream.close();
            pipedInputStream.close();
          } catch (IOException e) {
            e.printStackTrace();
          }
        }
      }

  }

  public static void main(String[] args) {
    ProducerConsumer05 producerConsumer05 = new ProducerConsumer05();
    new Thread(producerConsumer05.new Producer()).start();
    new Thread(producerConsumer05.new Consumer()).start();
  }
}

```

## 简述 Spring AOP 的原理

### 什么是AOP

使用OOP面向对象编程有一些弊端，当需要为多个不具有继承关系的对象引入同一个公共行为时，例如日志、安全检测等，我们只有在每个对象里引入公共行为，这样程序中就产生了大量的重复代码，程序就不便于维护了。所以就有了一个面向对象编程的补充，即面向方面编程，AOP所关注的方向是横向的，不同于OOP的纵向。

### 特点

* 降低模块之间的耦合度
* 使系统容易扩展
* 更好的代码复用

### 相关概念

1.切面（Aspect）：一个关注点的模块化，这个关注点可能会横切多个对象。
2.连接点（Joinpoint）：程序执行过程中某一行为。
3.通知（Advice）：“切面”对于某个“连接点”所产生的动作。
4.切入点（Pointcut）：匹配连接点的断言，在AOP中通知和一个切入点表达式关联。
5.目标对象（Target Object）：被一个或者多个切面所通知的对象。
6.AOP代理（AOP Proxy） 在Spring AOP有两种代理方式，JDK动态代理和CGLIB代理。

### Spring AOP的原理

Spring AOP采用的是动态代理，在运行期间对业务方法进行增强，所以不会生成新类。对于动态代理技术

* Spring提供了两种方式来生成代理对象:JDKProxy和Cglib，具体使用哪种方式是由配置来决定。
* 默认的策略是如果目标类是接口，则使用JDK动态代理技术，否则使用Cglib来生成代理。

1.DK动态代理只能为接口创建动态代理实例，而不能对类创建动态代理。需要获得被目标类的接口信息（应用Java的反射技术），生成一个实现了代理接口的动态代理类（字节码），再通过反射机制获得动态代理类的构造函数，利用构造函数生成动态代理类的实例对象，在调用具体方法前调用invokeHandler方法来处理。
2.CGLib动态代理需要依赖asm包，把被代理对象类的class文件加载进来，修改其字节码生成子类。但是Spring AOP基于注解配置的情况下，需要依赖于AspectJ包的标准注解，但是不需要额外的编译以及AspectJ的织入器，而基于XML配置不需要。

## Java 线程池里的 arrayblockingqueue 与 linkedblockingqueue 的使用场景和区别

### 1.ArrayBlockingQueue和LinkedBlockingQueue的实现原理

**ArrayBlockingQueue**

- ArrayBlockingQueue是一个带有长度的阻塞队列，初始化的时候必须要指定队列长度，且指定长度之后不允许进行修改。

- ArrayBlockingQueue的原理就是使用一个可重入锁和这个锁生成的两个条件对象进行并发控制(classic two-condition algorithm: notEmpty, notFull):
    1.  若某线程(线程A)要取数据时，数组正好为空，则该线程会执行notEmpty.await()进行等待；当其它某个线程(线程B)向数组中插入了数据之后，会调用notEmpty.signal()唤醒“notEmpty上的等待线程”。此时，线程A会被唤醒从而得以继续运行。
    2.  若某线程(线程H)要插入数据时，数组已满，则该线程会它执行notFull.await()进行等待；当其它某个线程(线程I)取出数据之后，会调用notFull.signal()唤醒“notFull上的等待线程”。此时，线程H就会被唤醒从而得以继续运行。

**LinkedBlockingQueue**

1. LinkedBlockingQueue继承于AbstractQueue，它本质上是一个FIFO(先进先出)的队列。
2. LinkedBlockingQueue实现了BlockingQueue接口，它支持多线程并发。当多线程竞争同一个资源时，某线程获取到该资源之后，其它线程需要阻塞等待。
3. LinkedBlockingQueue是通过单链表实现的。
4. LinkedBlockingQueue在实现“多线程对竞争资源的互斥访问”时，对于“插入”和“取出(删除)”操作分别使用了不同的锁。
- putLock是插入锁，takeLock是取出锁；notEmpty是“非空条件”，notFull是“未满条件”。通过它们对链表进行并发控制。
-  对于插入操作，通过“插入锁putLock”进行同步,插入锁putLock和“条件notFull”相关联；
-  对于取出操作，通过“取出锁takeLock”进行同步, 取出锁takeLock和“条件notEmpty”相关。

### 2.ArrayBlockingQueue和LinkedBlockingQueue的区别

**队列中锁的实现不同**

    ArrayBlockingQueue实现的队列中的锁是没有分离的，即生产和消费用的是同一个锁；

    LinkedBlockingQueue实现的队列中的锁是分离的，即生产用的是putLock，消费是takeLock

**在生产或消费时操作不同**

    ArrayBlockingQueue实现的队列中在生产和消费的时候，是直接将枚举对象插入或移除的；

    LinkedBlockingQueue实现的队列中在生产和消费的时候，需要把枚举对象转换为Node<E>进行插入或移除，会影响性能

**队列大小初始化方式不同**

    ArrayBlockingQueue实现的队列中必须指定队列的大小；

    LinkedBlockingQueue实现的队列中可以不指定队列的大小，但是默认是Integer.MAX_VALUE
    
### 3.ArrayBlockingQueue和LinkedBlockingQueue的使用场景

**ArrayBlockingQueue**

- 分析:
    - 由于基于数组，容量固定所以不容易出现内存占用率过高，但是如果容量太小，取数据比存数据的速度慢，那么会造成过多的线程进入阻塞(也可以使用offer()方法达到不阻塞线程)
    - 由于存取共用一把锁，所以有高并发和吞吐量的要求情况下也不建议使用ArrayBlockingQueue。
- 使用场景 
    - 案例：人事系统中员工离职/变更后，其他依赖应用进行数据同步。
    - 适合变更操作不是非常频繁的场景，这样能有效防止线程阻塞。
    - 如果基本没有并发和吞吐量的要求，所以可以将数据存放到ArrayBlockingQueue中。

**LinkedBlockingQueue**

- 特征: 
    - LinkedBlockingQueue基于链表实现，队列容量默认Integer.MAX_VALUE存/取数据的操作分别拥有独立的锁，可实现存/取并行执行。

- 分析:
    - 基于链表，数据的新增和移除速度比数组快，但是每次存储/取出数据都会有Node对象的新建和移除，所以也存在由于GC影响性能的可能
    - 默认容量非常大，所以存储数据的线程基本不会阻塞，但是如果消费速度过低，内存占用可能会飙升。
    - 读/取操作锁分离，所以适合有并发和吞吐量要求的项目中
- 使用场景:
    - 在项目的一些核心业务且生产和消费速度相似的场景中: 订单完成的邮件/短信提醒。
    - 如果订单的成交量非常大，那么使用ArrayBlockingQueue就会有一些问题，固定数组很容易被使用完，此时调用的线程会进入阻塞，那么可能无法及时将消息推送出去，所以使用LinkedBlockingQueue比较合适，但是要注意消费速度不能太低，不然很容易内存被使用完

## Java中sleep()和wait()方法的区别

### 1.Java sleep() vs wait() – 转换流程

**sleep()**

1. 调用 sleep 会让当前线程从 Running 进入 Timed Waiting 状态(阻塞)
2. 其它线程可以使用 interrupt 方法打断正在睡眠的线程，这时 sleep 方法会抛出 InterruptedException 
3. 睡眠结束后的线程未必会立刻得到执行

**wait()**

1. wait() 让进入 object 监视器的线程到 waitSet 等待变为Waiting/Timed Waiting状态(阻塞)
1. Waiting的线程会在线程调用notify和notifyAll时唤醒，但唤醒后并不一定会获得锁


### 2.Java sleep() vs wait() – 区别总结

1. sleep 是 Thread 方法，而 wait 是 Object 的方法 
1. sleep 不需要强制和 synchronized 配合使用，但 wait 需要 和 synchronized 一起用 
2. sleep 在睡眠的同时，不会释放对象锁的，但 wait 在等待的时候会释放对象锁 

## LinkedHashMap的原理，如何实现LRU

### 实现原理

- 1.LinkedHashMap继承自HashMap，所以它的底层仍然是基于拉链式散列结构。该结构由数组和链表+红黑树在此基础上LinkedHashMap增加了一条双向链表，保持遍历顺序和插入顺序一致的问题。
- 2.在实现上，LinkedHashMap很多方法直接继承自HashMap（比如putremove方法就是直接用的父类的），仅为维护双向链表覆写了部分方法（get（）方法是重写的）。
- 3.LinkedHashMap使用的键值对节点是Entity他继承了hashMap的Node,并新增了两个引用，分别是before和after。这两个引用的用途不难理解，也就是用于维护双向链表.
- 4.链表的建立过程是在插入键值对节点时开始的，初始情况下，让LinkedHashMap的head和tail引用同时指向新节点，链表就算建立起来了。随后不断有新节点插入，通过将新节点接在tail引用指向节点的后面，即可实现链表的更新
- 5.LinkedHashMap允许使用null值和null键，线程是不安全的，虽然底层使用了双线链表，但是增删相快了。因为他底层的Entity保留了hashMapnode的next属性。
- 6.如何实现迭代有序？
	重新定义了数组中保存的元素Entry（继承于HashMap.node)，该Entry除了保存当前对象的引用外，还保存了其上一个元素before和下一个元素after的引用，从而在哈希表的基础上又构成了双向链接列表。仍然保留next属性，所以既可像HashMap一样快速查找，
	用next获取该链表下一个Entry，也可以通过双向链接，通过after完成所有数据的有序迭代.
- 7.竟然inkHashMap的put方法是直接调用父类hashMap的，但在HashMap中，put方法插入的是HashMap内部类Node类型的节点，该类型的节点并不具备与LinkedHashMap内部类Entry及其子类型节点组成链表的能力。那么，LinkedHashMap是怎样建立链表的呢？
	虽然linkHashMap调用的是hashMap中的put方法，但是linkHashMap重写了，了一部分方法，其中就有newNode(inthash,Kkey,Vvalue,Node<K,V>e)linkNodeLast(LinkedHashMap.Entry<K,V>p)
	这两个方法就是第一个方法就是新建一个linkHasnMap的Entity方法，而linkNodeLast方法就是为了把Entity接在链表的尾部。
- 8.链表节点的删除过程
	与插入操作一样，LinkedHashMap删除操作相关的代码也是直接用父类的实现，但是LinkHashMap重写了removeNode()方法afterNodeRemoval（）方法，该removeNode方法在hashMap删除的基础上有调用了afterNodeRemoval回调方法。完成删除。
	根据hash定位到桶位置，遍历链表或调用红黑树相关的删除方法，从LinkedHashMap维护的双链表中移除要删除的节点

### LRU

- 最近最久未使用策略，优先淘汰最久未使用的数据，也就是上次被访问时间距离现在最久的数据。该策略可以保证内存中的数据都是热点数据，也就是经常被访问的数据，从而保证缓存命中率。

- LinkedHashMap中本身就实现了一个方法removeEldestEntry用于判断是否需要移除最不常读取的数，方法默认是直接返回false，不会移除元素，所以需要【重写该方法】，即当缓存满后就移除最不常用的数。
	设定最大缓存空间 MAX_ENTRIES 为 3；
	使用 LinkedHashMap 的构造函数将 accessOrder 设置为 true，开启 LRU 顺序；
	覆盖 removeEldestEntry() 方法实现，在节点多于 MAX_ENTRIES 就会将最近最久未使用的数据移除

## Spring中bean的生命周期

- 1.实例化一个Bean
- 2.按照Spring上下文对实例化的Bean进行配置
- 3.如果这个Bean已经实现了BeanNameAware接口，会调用它实现的setBeanName(String)方法，此处传递的就是Spring配置文件中Bean的id值
- 4.如果这个Bean已经实现了BeanFactoryAware接口，会调用它实现的setBeanFactory(setBeanFactory(BeanFactory)传递的是Spring工厂自身
- 5.如果这个Bean已经实现了ApplicationContextAware接口，会调用setApplicationContext(ApplicationContext)方法，传入Spring上下文
- 6.如果这个Bean关联了BeanPostProcessor接口，将会调用postProcessBeforeInitialization(Objectobj,Strings)方法，BeanPostProcessor经常被用作是Bean内容的更改，并且由于这个是在Bean初始化结束时调用那个的方法，也可以被应用于内存或缓存技术
- 7.如果Bean在Spring配置文件中配置了init-method属性会自动调用其配置的初始化方法。
- 8.如果这个Bean关联了BeanPostProcessor接口，将会调用postProcessAfterInitialization(Objectobj,Strings)方法.；
- 9.当Bean不再需要时，会经过清理阶段，如果Bean实现了DisposableBean这个接口，会调用那个其实现的destroy()方法；
- 10.如果这个Bean的Spring配置中配置了destroy-method属性，会自动调用其配置的销毁方法。

## 简述 Java 中的自动装箱与拆箱

### 一. 什么是自动装箱和拆箱
1. 自动装箱：Java自动将原始类型值转换成对应的对象，如将int的变量转换成Integer对象。
2. 自动拆箱: 将Integer对象转换成int类型值。
3. 因为这里的装箱和拆箱是自动进行的非人为转换，所以就称作为自动装箱和拆箱。

##### 有哪些类型？
原始类型byte,short,char,int,long,float,double和boolean对应的封装类为Byte,Short,Character,Integer,Long,Float,Double,Boolean。


### 二.如何实现 
1. 自动装箱时编译器调用valueOf将原始类型值转换成对象。
1. 自动拆箱时，编译器通过调用类似intValue(),doubleValue()这类的方法将对象转换成原始类型值。


### 三. 实践
- 因为基础数据类型的优点是会缓存一些基本的数据类型值，所以效率更高，一般建议尽量使用基本数据类型。
- 使用缓存的策略是因为缓存的这些对象都是经常使用到的，防止每次自动装箱都创建一次对象的实例。
- 其中只有double和float没有使用缓存，每次都是new一个新对象。

## SpringBoot 是如何进行自动配置的？

1. 在使用main()启动SpringBoot的时候，只有一个注解@SpringBootApplication;
2. @SpringBootApplication等同于下面三个注解：

- @SpringBootConfiguration: 底层是Configuration注解,就是支持JavaConfig的方式来进行配置(使用Configuration配置类等同于XML文件)。
- @EnableAutoConfiguration: 开启自动配置功能
- @ComponentScan: 扫描注解，默认是扫描当前类下的package。将@Controller/@Service/@Component/@Repository等注解加载到IOC容器中。

其中@EnableAutoConfiguration是关键(启用自动配置)，内部实际上就去加载META-INF/spring.factories文件的信息，然后筛选出以EnableAutoConfiguration为key的数据，加载到IOC容器中，实现自动配置功能。

## 简述 HashSet 实现原理

### HashSet和HashMap的关系

- HashSet中不允许有重复元素，这是因为HashSet是基于HashMap实现的
- HashSet中的元素都存放在HashMap的key上面，而value中的值都是统一的一个private static final Object PRESENT = new Object()。
- HashSet跟HashMap一样，都是一个存放链表的数组。

### HashSet的组成结构

- HashSet是基于HashMap实现的，HashSet底层使用HashMap来保存所有元素。
- HashSet中的元素，只是存放在了底层HashMap的key上， 而value使用一个static final的Object对象标识。
- 因此HashSet 的实现比较简单，相关HashSet的操作，基本上都是直接调用底层HashMap的相关方法来完成。

### 总结

- HashSet底层由HashMap实现
- HashSet的值存放于HashMap的key上
- HashMap的value统一为PRESENT

## 高并发情景下，核心线程池该如何设置参数？

### 线程数设置多少合适

创建多少线程合适，要看多线程具体的应用场景。I/O 密集型程序和 CPU密集型程序，计算最佳线程数的方法是不同的。

1. CPU 密集型计算:

- 多线程本质上是提升多核 CPU 的利用率，所以对于一个 4 核的 CPU，每个核一个线程，理论上创建 4 个线程就可以了，再多创建线程也只是增加线程切换的成本。
- 理论上“线程的数量 = CPU 核数”就是最合适的。
- 不过在工程上，线程的数量一般会设置为“CPU 核数 +1”，这样的话，当线程因为偶尔的内存页失效或其他原因导致阻塞时，这个额外的线程可以顶上，从而保证 CPU 的利用率。

2. I/O 密集型的计算场景

- 对于单个CPU可以总结出这样一个公式：最佳线程数 =1 +（I/O 耗时 / CPU 耗时）
- 对于多核 CPU，需要等比扩大，计算公式如下：最佳线程数 =CPU 核数 * [ 1 +（I/O 耗时 / CPU 耗时）]

3. 实际应用：

- 在不同的业务场景以及不同配置的部署机器中，线程池的线程数量设置是不一样的。
- 其设置不宜过大，也不宜过小，要根据具体情况，计算出一个大概的数值，再通过实际的性能测试，计算出一个合理的线程数量。
- 保证 CPU 处理线程的最大化
- **设置核心线程数的时候，同时设置最大线程数即可。其实可以把二者设置为相同的值**

### 禁止使用Executors方法的原因

- 禁止使用Executors方来创建线程池，而应该手动 new ThreadPoolExecutor 来创建线程池。
- 最重要的原因是：Executors 提供的很多方法默认使用的都是**无界的** LinkedBlockingQueue，高负载情境下，无界队列很容易导致 **OOM**，而 OOM 会导致所有请求都无法处理，这是致命问题。
- 最典型的就是 newFixedThreadPool 和 newCachedThreadPool，可能因为资源耗尽导致 OOM 问题。