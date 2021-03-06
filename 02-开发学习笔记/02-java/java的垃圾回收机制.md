垃圾回收机制是 Java 非常重要的特性之一，也是面试题的常客。它让开发者无需关注空间的创建和释放，而是以守护进程的形式在后台自动回收垃圾。这样做不仅提高了开发效率，更改善了内存的使用状况。

今天本文来对垃圾回收机制进行讲解，主要涉及下面几个问题：

- 什么是堆内存？
- 什么是垃圾？
- 有哪些方法回收这些垃圾？
- 什么是分代回收机制？

## 什么是 Java 堆内存

堆是在 JVM 启动时创建的，主要用来维护运行时数据，如运行过程中创建的对象和数组都是基于这块内存空间。Java 堆是非常重要的元素，如果我们动态创建的对象没有得到及时回收，持续堆积，最后会导致堆空间被占满，内存溢出。

因此，Java 提供了一种垃圾回收机制，在后台创建一个守护进程。该进程会在内存紧张的时候自动跳出来，把堆空间的垃圾全部进行回收，从而保证程序的正常运行。

## 那什么是垃圾呢？

所谓“垃圾”，就是指所有不再存活的对象。常见的判断是否存活有两种方法：引用计数法和可达性分析。

### 引用计数法

为每一个创建的对象分配一个引用计数器，用来存储该对象被引用的个数。当该个数为零，意味着没有人再使用这个对象，可以认为“对象死亡”。但是，这种方案存在严重的问题，就是无法检测“循环引用”：当两个对象互相引用，即时它俩都不被外界任何东西引用，它俩的计数都不为零，因此永远不会被回收。而实际上对于开发者而言，这两个对象已经完全没有用处了。

因此，Java 里没有采用这样的方案来判定对象的“存活性”。

### 可达性分析

这种方案是目前主流语言里采用的对象存活性判断方案。基本思路是把所有引用的对象想象成一棵树，从树的根结点 GC Roots 出发，持续遍历找出所有连接的树枝对象，这些对象则被称为“可达”对象，或称“存活”对象。其余的对象则被视为“死亡”的“不可达”对象，或称“垃圾”。

参考下图，object5,object6和object7便是不可达对象，视为“死亡状态”，应该被垃圾回收器回收。

![image-20200119174619342](/Users/baola/Library/Application%20Support/typora-user-images/image-20200119174619342.png)

#### GC Roots 究竟指谁呢？

我们可以猜测，GC Roots 本身一定是可达的，这样从它们出发遍历到的对象才能保证一定可达。那么，Java 里有哪些对象是一定可达呢？主要有以下四种：

- 虚拟机栈（帧栈中的本地变量表）中引用的对象。
- 方法区中静态属性引用的对象。
- 方法区中常量引用的对象。
- 本地方法栈中JNI引用的对象。

不少读者可能对这些 GC Roots 似懂非懂，这涉及到 JVM 本身的内存结构等等，未来的文章会再做深入讲解。这里只要知道有这么几种类型的 GC Roots，每次垃圾回收器会从这些根结点开始遍历寻找所有可达节点。

## 有哪些方式来回收这些垃圾呢？

上面已经知道，所有GC Roots不可达的对象都称为垃圾，参考下图，黑色的表示垃圾，灰色表示存活对象，绿色表示空白空间。

![image-20200119174655885](/Users/baola/Library/Application%20Support/typora-user-images/image-20200119174655885.png)

那么，我们如何来回收这些垃圾呢？

#### 标记－清理

第一步，所谓“标记”就是利用可达性遍历堆内存，把“存活”对象和“垃圾”对象进行标记，得到的结果如上图； 第二步，既然“垃圾”已经标记好了，那我们再遍历一遍，把所有“垃圾”对象所占的空间直接`清空`即可。

结果如下：

![image-20200119174721364](/Users/baola/Library/Application%20Support/typora-user-images/image-20200119174721364.png)

这便是`标记－清理`方案，`简单方便`，但是容易产生`内存碎片`。

#### 标记－整理

既然上面的方法会产生内存碎片，那好，我在清理的时候，把所有`存活`对象扎堆到同一个地方，让它们待在一起，这样就没有内存碎片了。

结果如下：

![image-20200119174752345](/Users/baola/Library/Application%20Support/typora-user-images/image-20200119174752345.png)

这两种方案适合`存活对象多，垃圾少`



#### 复制

这种方法比较粗暴，直接把堆内存分成两部分，一段时间内只允许在其中一块内存上进行分配，当这块内存被分配完后，则执行垃圾回收，把所有`存活`对象全部复制到另一块内存上，当前内存则直接全部清空。

参考下图：

![image-20200119174815838](/Users/baola/Library/Application%20Support/typora-user-images/image-20200119174815838.png)

起初时只使用上面部分的内存，直到内存使用完毕，才进行垃圾回收，把所有存活对象搬到下半部分，并把上半部分进行清空。

这种做法不容易产生碎片，也简单粗暴；但是，它意味着你在一段时间内只能使用一部分的内存，超过这部分内存的话就意味着堆内存里频繁的`复制清空`。

这种方案适合`存活对象少，垃圾多`的情况，这样在复制时就不需要复制多少对象过去，多数垃圾直接被清空处理。