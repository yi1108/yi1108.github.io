---
title: Java的IO模型
date: 2024-02-05 17:00:07
tags:
- nio
---

#### 文章目录

*   [二、Java的IO模型](#JavaIO_5)
*   *   [2.1 Java的IO模型支持](#21_JavaIO_7)
    *   [2.2 BIO（blocking I/O）模型](#22_BIOblocking_IO_17)
    *   [2.3 NIO（non-blocking I/O）模型](#23_NIOnonblocking_IO_34)
    *   *   [2.3.1 Buffer（缓冲区）](#231_Buffer_58)
        *   [2.3.2 Channel（通道）](#232_Channel_73)
        *   [2.3.3 Selector（选择器）](#233_Selector_86)
    *   [2.3 AIO模型](#23_AIO_92)
    *   [2.4 JavaIO模型小结](#24_JavaIO_102)
*   [](#_108)

* * *

二、Java的[IO模型](https://so.csdn.net/so/search?q=IO%E6%A8%A1%E5%9E%8B&spm=1001.2101.3001.7020)
-------------------------------------------------------------------------------------------

### 2.1 Java的IO模型支持

Java共支持3种网络编程模型/IO模式，分别是：

*   **BIO（同步阻塞IO）**
*   **NIO（同步非阻塞IO）**
*   **AIO（异步非阻塞IO）**

我们可以根据不同的业务场景来决定选择不同I/O处理模型；

### 2.2 [BIO](https://so.csdn.net/so/search?q=BIO&spm=1001.2101.3001.7020)（blocking I/O）模型

*   **`BIO（blocking I/O）`：也叫同步阻塞IO**，在JDK1.4之前，我们建立网络连接的时候采用的是 BIO 模式。 阻塞 IO（BIO）是最传统的一种 IO 模型，即在读写数据过程中会发生阻塞现象，直至有可供读取的数据或者数据能够写入。

在BIO模式中，服务器会为每个客户端的请求创建一个对应的线程来处理，由该线程单独负责处理一个客户请求，如果这个连接不做任何事情会造成不必要的线程开销，为此我们可以通过线程池机制改善性能；

![在这里插入图片描述](https://img-blog.csdnimg.cn/17fb86a8f4a042c6b00cdb516e4f2990.png#pic_center)

虽然使用线程池可以一定程度上改善BIO的性能，但依旧无法冲本质上解决BIO同步阻塞的的问题；如果连接大多是长连接，则会导致连接无法释放，新的请求将无法得到处理，另外，BIO这种一个请求对应一个线程的方式在应对高并发的情况下，服务器必须也要创建同等量的线程来处理客户端的请求，这样对系统的消耗是非常大的；

*   BIO优点：
    *   实现简单，IO模式适用于连接数目比较小且固定的架构，是JDK1.4以前的唯一选择；
*   BIO缺点：
    *   1）每个请求都需要创建独立的线程来处理，当连接数较大时，需要创建大量的线程来处理
    *   2）一个线程只能处理一个请求，连接建立后，如果当前线程暂时没有数据可读，那么该线程就一直阻塞在读操作，并不能处理其他事情，造成性能浪费；

### 2.3 [NIO](https://so.csdn.net/so/search?q=NIO&spm=1001.2101.3001.7020)（non-blocking I/O）模型

NIO是从JDK1.4版本开始引入的一个新的IO API，NIO支持面**向缓冲区**的、基于**通道**的IO操作。NIO将以更加高效的方式进行文件的读写操作。

*   **BIO是同步阻塞IO**，同步：即在同一时间点只能同时处理一个客户端连接，阻塞：即当调用方法获取数据时，如果没有可用的数据将会阻塞当前线程；
    
*   **NIO则是同步非阻塞IO**，NIO中有三大组件，分别是：Channel（通道），Buffer（缓冲区），Selector（选择器）；当有客户端连接时，服务器可以获取与该客户端的连接Channel（通道），所有的通道都会被注册到Selector（选择器）上，当Channel上有读写数据时将会被Selector侦测到，服务器只需要派发一个线程来处理Selector上的事件即可；当前Channel如果没有读写数据时，Selector并不会一直阻塞的等待Channel的数据返回，而是轮询式的侦测所有的Channel，**这是NIO非阻塞的核心；**
    

另外，客户端的连接都变成了Channel，这些Channel都注册到了Selector中，服务器再也不需要为每一个连接来创建一个独立的线程为之服务了；**这也是NIO能够应对高并发的核心之一；**

*   **`NIO（non-blocking IO）`**：也叫**同步非阻塞IO**，由于BIO的各种弊端，JDK1.4从开始提供了一系列改进的输入/输出的新特性，被统称为 NIO(即 New IO)，是同步非阻塞的。NIO相关类都被放在`java.nio`包及子包下，并且对原`java.io`包中的很多类进行改写。

* * *

在NIO模型中，每个请求都会有一个与服务器做数据交互的通道（Channel），所有的通道都被注册到一个选择器中（selector），当需要与服务器做数据交互时，数据通过管道写入到一个缓冲区（Buffer）中，服务器通过往缓冲区中读取数据，如果当前通道没有数据时，就什么都不会获取，**而不是保持线程阻塞**，直至数据变的可以读取之前，该线程可以继续做其他的事情。

![在这里插入图片描述](https://img-blog.csdnimg.cn/232f1b5929454fa1b4c9867677381e3c.png#pic_center)

> **在Java NIO有三大核心部分：Buffer（缓冲区）、Channel（通道）、Selector（选择器） ；**

#### 2.3.1 Buffer（缓冲区）

**Buffer本质上就是一块存储数据的内存**，我们可以在这一块内存中进行读写操作，这与我们之前的数组非常类似。与数组不同的是，这块内存被封装成Buffer对象，并根据不同的数据类型提供有不同的Buffer子类。Java对Buffer提供了更多的API，使得Buffer功能更加强大；

*   Java中常见的Buffer如下：
    *   CharBuffer
    *   DoubleBuffer
    *   IntBuffer
    *   LongBuffer
    *   ByteBuffer
    *   ShortBuffer
    *   FloatBuffer

> Tips：以上Buffer都继承与Buffer抽象类，StringBuffer和以上的Buffer并不是同一类的，没有继承与NIO包下的Buffer接口；

#### 2.3.2 Channel（通道）

Java NIO的通道类似流，都是用于传输数据的。但通过又与流有些不同；**流的数据走向是单向的**，分为输入流（只能读取数据），输出流（只能写出数据），但NIO中的通道不一样，**通道既可以写数据到Buffer，又可以从Buffer中读取数据；**

**另外流的操作对象是数组，而通道的操作对象是Buffer；**

*   Java中常见的Channel如下：
    *   `FileChannel`：用于文件 I/O 编程
    *   `SocketChannel`、`ServerSocketChannel`：用于 TCP I/O 编程
    *   `DatagramChannel`：用于 UDP I/O 编程

#### 2.3.3 Selector（选择器）

Selector选择器，也叫多路复用器；**NIO中实现非阻塞 I/O 的核心对象就是 Selector**。当一个连接创建后，不需要创建一个线程来处理这个来连接，这个连接（管道）会被注册到选择器上，选择器可以检查一个或多个 NIO 通道，并确定哪些通道已经准备好进行读取或写入。这样，一个单独的线程可以管理多个channel，系统不必创建大量的线程，也不必维护这些线程，从而大大减小了系统的开销。

### 2.3 AIO模型

*   **`AIO`** ：AIO也叫**异步非阻塞**，JDK1.7之后的新特性，AIO引入异步通道的概念，采用了 Proactor 模式，简化了程序编写，有效的请求才启动线程，它的特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连接时间较长的应用。

与NIO模型不同，读写操作为例，只需直接调用read和write的API即可，这方法都是异步的对于读操作:当有流可读是，系统会将可读的流传入到read方法的缓冲区，并通知应用程序读写都是异步的，完成之后会主动调用回调函数

AIO需要操作系统的支持，在Linux内核2.6版本之后增加了对真正异步IO的实现。Java从JDK1.7之后支持AIO，JDK1.7新增一些与文件/网络IO相关的一些API，称之为NIO2.0或者称之为AIO（Asynchronous IO）。AIO最大的特征提供了异步功能，对于socket网络通信和文件IO都是起作用的。

目前 AIO 还没有广泛应用，Netty也是基于NIO，而不是AIO。

### 2.4 JavaIO模型小结

*   BIO方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4以前的唯一选择，但程序直观简单易理解；
*   NIO方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂，JDK1.4开始支持；
*   AIO方式使用于连接数目多且连接比较长（重操作）的架构，比如文件服务器，充分调用OS参与并发操作，编程比较复杂，JDK7开始支持。

 

仅限学习、交流使用！

![](https://g.csdnimg.cn/extension-box/1.1.6/image/qq.png) QQ群名片

![](https://g.csdnimg.cn/extension-box/1.1.6/image/ic_move.png)

本文转自 <https://blog.csdn.net/Bb15070047748/article/details/125438283>，如有侵权，请联系删除。