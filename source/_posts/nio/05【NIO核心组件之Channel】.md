---
title: 05 NIO核心组件之Channel
date: 2024-02-05 17:00:07
tags:
- nio
--- 

#### 文章目录

*   [五、NIO核心组件之Channel](#NIOChannel_4)
*   *   [5.1 FileChannel](#51_FileChannel_17)
    *   *   [5.1.1 FileChannel的基本使用](#511_FileChannel_37)
        *   [5.1.2 FileChannel的常用方法](#512_FileChannel_251)
        *   *   [1）truncate](#1truncate_265)
            *   [2）size](#2size_304)
            *   [3）position](#3position_323)
            *   [4）force](#4force_408)
            *   [5）transferFrom](#5transferFrom_442)
            *   [6）transferTo](#6transferTo_466)
        *   [5.1.3 聚集和分散](#513__492)
    *   [5.2 DatagramChannel](#52_DatagramChannel_582)
    *   *   [5.2.1 DatagramChannel简介](#521_DatagramChannel_584)
        *   [5.2.2 DatagramChannel的获取](#522_DatagramChannel_598)
        *   [5.2.3 DatagramChannel的常用方法](#523_DatagramChannel_655)
        *   *   [1）发送和接收](#1_667)
            *   [2）读取和写入](#2_730)
    *   [5.3 SocketChannel与ServerSocketChannel](#53_SocketChannelServerSocketChannel_930)
    *   *   [5.3.1 Channel的简介](#531_Channel_932)
        *   [5.3.2 Channel的获取](#532_Channel_938)
        *   [5.3.3 Channel的常用方法](#533_Channel_1020)
        *   *   [1）基本读写](#1_1039)
            *   [2）拷贝图片案例](#2_1121)
            *   [3）在线聊天案例](#3_1125)

* * *

五、[NIO](https://so.csdn.net/so/search?q=NIO&spm=1001.2101.3001.7020)核心组件之Channel
--------------------------------------------------------------------------------

Java NIO的通道类似流，都是用于传输数据的。但通过又与流有些不同；**流的数据走向是单向的**，分为输入流（只能读取数据），输出流（只能写出数据），但NIO中的通道不一样，**通道既可以写数据到Buffer，又可以从Buffer中读取数据；**

**另外流的操作对象是数组，而通道的操作对象是Buffer；**

*   Java中常见的Channel如下：
    *   `FileChannel`：用于文件 I/O 编程
    *   `SocketChannel`、`ServerSocketChannel`：用于 TCP I/O 编程
    *   `DatagramChannel`：用于 UDP I/O 编程

### 5.1 FileChannel

FileChannel是用于文件I/O编程的管道类；通过FileInputStream和FileOutputStream可以获取一个关联文件的Channel，即FileChannel；

*   `public FileChannel getChannel()`：获取该流的Channel；

示例代码：

```java
// 通过输入流获取Channel,该Channel只能读取数据
FileChannel inChannel = new FileInputStream("").getChannel();

// 通过输出流获取Channel,该Channel只能写出数据
FileChannel outChannel = new FileOutputStream("").getChannel();
```

**我们可以读取Channel中的数据，也可以往Channel中写数据，Channel是可读可写的；但FileChannel最终需要将数据写入到对应的输入/输出流，因为流是有顺序的，输入流只能读取数据，而输出流只能写出数据，因此使用FileInputStream/FileOutputStream获取到的FileChannel只能读或写；**

#### 5.1.1 FileChannel的基本使用

*   `int read(ByteBuffer dst)`：从Channel中读取数据，写入到Buffer中，返回读取到的有效字节个数，读取到末尾返回-1
*   `int write(ByteBuffer src)`：从Buffer中读取数据，写入到Channel中，返回写出的有效字节个数，如果没有数据写出返回0

案例1：通过Channel输出数据

```java
package com.dfbz.channel.fileChannel;

import org.junit.Test;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @author lscl
 * @version 1.0
 * @intro:
 */
public class Demo01_FileChannel的基本使用 {

    @Test
    public void writer() throws Exception {

        // 1. 创建一个输出流
        FileOutputStream fos = new FileOutputStream("001.txt");

        // 2. 通过输出流获取Channel
        FileChannel channel = fos.getChannel();

        // 3. 创建一个Buffer
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        buffer.put("hello".getBytes());

        // 4. 切换读模式(limit=position=5;position=0)
        buffer.flip();

        // 5. 将Buffer的数据读出来,写入到Channel中
        int len = channel.write(buffer);            //

        System.out.println("写出的有效字节个数: " + len);           // 5
        len = channel.write(buffer);
        System.out.println("写出的有效字节个数: " + len);           // 0

        channel.close();
        fos.close();
    }
}

```

**注意：channel.write()方法是将数据从Buffer中读取出来，然后写入到Channel中，这对Buffer本质上是一次读操作，我们对Buffer的任何读写操作都会造成Buffer中的position位移；**

再次测试：

```java
@Test
public void writer_02() throws Exception {

    // 1. 创建一个输出流
    FileOutputStream fos = new FileOutputStream("001.txt");

    // 2. 通过输出流获取Channel
    FileChannel channel = fos.getChannel();

    // 3. 创建一个Buffer
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    System.err.println(buffer);             // position=0,limit=capacity=1024
    buffer.put("hello".getBytes());
    System.err.println(buffer);             // position=5,limit=capacity=1024

    // 4. 切换读模式(limit=position=5;position=0)
    buffer.flip();
    System.err.println(buffer);             // position=0,limit=5,capacity=1024


    // 5. 将Buffer的数据读出来(也会造成position的位移),写入到Channel中
    int len = channel.write(buffer);            //

    System.out.println("写出的有效字节个数: " + len);           // 5
    len = channel.write(buffer);
    System.out.println("写出的有效字节个数: " + len);           // 0

    System.err.println(buffer);             // position=5,limit=5,capacity=1024

    channel.close();
    fos.close();
}
```

执行程序，查看控制台：

![在这里插入图片描述](https://img-blog.csdnimg.cn/d273d59ba9a5402c81bb5bdafbb28d0b.png#pic_center)

案例2：通过Channel读取数据

```java
@Test
public void reader() throws Exception {

    // 1. 创建一个输入流
    FileInputStream fis = new FileInputStream("001.txt");

    // 2. 通过输入流获取Channel
    FileChannel channel = fis.getChannel();

    // 3. 创建一个Buffer
    ByteBuffer buffer = ByteBuffer.allocate(1024);

    // 4. 从Channel中读取数据,写入到Buffer中
    int len = channel.read(buffer);

    System.out.println("读取到的有效字节个数: " + len);           // 5
    len = channel.read(buffer);
    System.out.println("读取到的有效字节个数: " + len);           // -1

    // limit=position,position=0
    buffer.flip();

    byte[] data = new byte[buffer.limit()];
    buffer.get(data);           // 从Buffer中读取数据

    System.out.println(new String(data, 0, data.length));

    channel.close();
    fis.close();
}
```

**注意：和write方法一样，channel调用read方法是将数据从Channel读取出来，往Buffer中写入，这对Buffer来说本质上是一种写的操作，我们对Buffer的任何读写操作都会造成Buffer中的position位移；**

再次测试：

```java
@Test
public void reader_01() throws Exception {

    // 1. 创建一个输入流
    FileInputStream fis = new FileInputStream("001.txt");       // 文件内容: hello

    // 2. 通过输入流获取Channel
    FileChannel channel = fis.getChannel();

    // 3. 创建一个Buffer
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    System.err.println(buffer);                                 // position=0,limit=capacity=1024

    // 4. 从Channel中读取数据,写入到Buffer中(会造成position的位移)
    channel.read(buffer);
    System.err.println(buffer);                                 // position=5,limit=capacity=1024

    // limit=position=5,position=0,capacity=1024
    buffer.flip();
    System.err.println(buffer);                                 // position=0,limit=5,capacity=1024

    byte[] data = new byte[buffer.limit()];
    // 从Buffer中读取数据
    buffer.get(data);
    System.err.println(buffer);                                 // position=5,limit=5,capacity=1024

    System.out.println(new String(data, 0, data.length));

    channel.close();
    fis.close();
}
```

执行程序，查看控制台：

![在这里插入图片描述](https://img-blog.csdnimg.cn/cfd36769d45844d8b502bfd17e0d60a3.png#pic_center)

*   案例3-使用FileChannel拷贝文件：

```java
@Test
public void copy() throws Exception {
    FileInputStream fis = new FileInputStream("100.png");
    FileOutputStream fos = new FileOutputStream("200.png");

    FileChannel inChannel = fis.getChannel();
    FileChannel outChannel = fos.getChannel();

    ByteBuffer buffer = ByteBuffer.allocate(1024);

    // 读取Channel中的数据,写入到Buffer中
    while (inChannel.read(buffer) != -1) {

        // limit=position,position=0
        buffer.flip();

        // 从Buffer中读取数据,写入Channel中
        outChannel.write(buffer);

        // limit=capacity,position=0
        buffer.clear();
    }

    inChannel.close();
    outChannel.close();

    fis.close();
    fos.close();
}
```

#### 5.1.2 FileChannel的常用方法

*   `FileChannel truncate(long s)`：将此通道的文件截取为给定大小
*   `long size()`：返回此通道的文件的当前大小
*   `long position()`：返回该Channel目前所在的文件位置；
*   `FileChannel position(long p)`：设置该Channel目前所在的文件位置；
*   `public void force(boolean metaData)`：将当前Channel中的数据强制写入到磁盘中
*   `public long transferFrom(ReadableByteChannel src, long position, long count)`：从src的position位置开始读取，读取count个字节到当前Channel中
*   `public long transferTo(long position, long count,WritableByteChannel target)`：从当前Channel的position位置开始读取，读取count个字节到target中
*   `long write(ByteBuffer[] srcs)` 将ByteBuffer\[\]到中的数据全部写入（聚集）到 Channel
*   `long read(ByteBuffer[] dsts)` 将Channel到中的数据读取出来，然后全部写入（分散）到ByteBuffer\[\]

* * *

##### 1）truncate

*   `FileChannel truncate(long s)`：将此通道的文件截取为给定大小；

可以使用FileChannel.truncate()方法截取一个文件。截取文件时，文件将中指定长度后面的部分将被删除。

```java
@Test
public void truncate() throws Exception {

    FileInputStream stream = new FileInputStream("001.txt");                     // 文件内容: hello
//        FileOutputStream stream = new FileOutputStream("001.txt");                        // 文件内容: hello

    // 创建流(可读可写)
//        RandomAccessFile stream = new RandomAccessFile("001.txt", "rw");       // 文件内容: hello

    // 获取Channel
    FileChannel channel = stream.getChannel();

    // 将Channel的文件截取为2
    channel.truncate(2);                            // 使用FileInputStream出现: java.nio.channels.NonWritableChannelException

    ByteBuffer buffer = ByteBuffer.allocate(10);

    channel.read(buffer);                           // 使用FileOutputStream出现: java.nio.channels.NonReadableChannelException

    System.out.println(new String(buffer.array(), 0, buffer.array().length));         // he

    stream.close();
    channel.close();
}
```

> Tips：FileInputStream只能读取数据，因此使用FileInputStream获取的FileChannel也只能读取数据；同理FileOutputStream只能写出数据，使用FileOutputStream获取FileChannel也只能写出数据；因此上述案例中使用`RandomAccessFile`，即可读又可写；

##### 2）size

FileChannel实例的size()方法将返回该实例所关联文件的大小。

*   `long size()`：返回此通道的文件的当前大小

```java
@Test
public void size() throws Exception {

    FileInputStream fis = new FileInputStream("001.txt");
    FileChannel channel = fis.getChannel();

    System.out.println(channel.size());         // 5
}
```

##### 3）position

*   `long position()`：返回该Channel目前所在的文件位置；

有时可能需要在FileChannel的某个特定位置进行数据的读/写操作。可以通过调用position()方法获取FileChannel的当前位置。也可以通过调用position(long pos)方法设置FileChannel的当前位置。

*   测试position：

```java
package com.dfbz.channel;

import org.junit.Test;

import java.io.FileInputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @author lscl
 * @version 1.0
 * @intro:
 */
public class Demo02_常用方法 {
    @Test
    public void position() throws Exception {

        FileInputStream fis = new FileInputStream("001.txt");
        FileChannel channel = fis.getChannel();

        // 获取此Channel的读写位置(默认为0)
        System.out.println(channel.position());         // 0

        ByteBuffer buffer = ByteBuffer.allocate(1024);

        // 从Channel中读取数据到buffer中,读取了5个有效字节
        channel.read(buffer);
        System.out.println("buffer中的内容: " + new String(buffer.array(), 0, buffer.array().length));

        System.out.println("---------------------");
        // limit=capacity,position=0
        buffer.clear();

        System.out.println(channel.position());         // 5

        // 将channel的读取位置设置为2
        channel.position(2);
        channel.read(buffer);
        System.out.println("buffer中的内容: " + new String(buffer.array(), 0, buffer.array().length));
    }
}
```

> Tips：如果将位置设置在文件结束符之后，然后试图从文件通道中读取数据，读方法将返回-1 —— 文件结束标志。

**如果将位置设置在文件结束符之后，然后向通道中写数据，文件将撑大到当前位置并写入数据。这可能导致“文件空洞”，磁盘上物理文件中写入的数据间有空隙。**

*   测试position-2：

```java
@Test
public void position2() throws Exception {

    // 创建流(可读可写)
    RandomAccessFile raf = new RandomAccessFile("001.txt", "rw");       // 文件内容: hello

    // 获取Channel
    FileChannel channel = raf.getChannel();

    // 将position设置为size+10(空隙)
    channel.position(channel.size()+10);

    // 往Channel写出数据
    channel.write(ByteBuffer.wrap("abc".getBytes()));

    raf.close();
    channel.close();
}
```

001.txt文件内容如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/7e8a50750d994247ab3eeafaf802ee4f.png#pic_center)

##### 4）force

*   `public void force(boolean metaData)`：将当前Channel中的数据强制写入到磁盘中

FileChannel.force()方法将通道里尚未写入磁盘的数据强制写到磁盘上。出于性能方面的考虑，操作系统会将数据缓存在内存中，所以无法保证写入到FileChannel里的数据一定会即时写到磁盘上。要保证这一点，需要调用force()方法。

另外，force()方法有一个[boolean类型](https://so.csdn.net/so/search?q=boolean%E7%B1%BB%E5%9E%8B&spm=1001.2101.3001.7020)的参数，指明是否同时将文件元数据（权限信息等）写到磁盘上

*   测试代码：

```java
@Test
public void force() throws Exception {

    // 创建流(可读可写)
    RandomAccessFile raf = new RandomAccessFile("001.txt", "rw");       // 文件内容: hello

    // 获取Channel
    FileChannel channel = raf.getChannel();

    channel.write(ByteBuffer.wrap("Hello Everyone!".getBytes()));

    /*
        将内存中的数据强制写入到磁盘中
            true: 将文件元信息(权限信息等)写入磁盘
            false: 不写入文件元信息到磁盘
     */
    channel.force(true);

    raf.close();
    channel.close();
}
```

##### 5）transferFrom

*   `public long transferFrom(ReadableByteChannel src, long position, long count)`：从src的position位置开始读取，读取count个字节到当前Channel中

FileChannel的transferFrom()方法可以将数据从源通道传输到FileChannel中

```java
@Test
public void transferFrom() throws Exception {

    RandomAccessFile fromFile = new RandomAccessFile("001.txt", "rw");
    FileChannel fromChannel = fromFile.getChannel();

    RandomAccessFile toFile = new RandomAccessFile("002.txt", "rw");
    FileChannel toChannel = toFile.getChannel();

    long position = 0;
    long count = fromChannel.size();

    // 从fromChannel的position位置开始读取,读取count个字节到toChannel中
    toChannel.transferFrom(fromChannel, position, count);
}
```

##### 6）transferTo

*   `public long transferTo(long position, long count,WritableByteChannel target)`：从当前Channel的position位置开始读取，读取count个字节到target中

transferTo()方法将数据从FileChannel传输到其他的channel中

```java
@Test
public void transferTo() throws Exception {

    RandomAccessFile fromFile = new RandomAccessFile("001.txt", "rw");
    FileChannel fromChannel = fromFile.getChannel();

    RandomAccessFile toFile = new RandomAccessFile("002.txt", "rw");
    FileChannel toChannel = toFile.getChannel();

    long position = 0;
    long count = fromChannel.size();

    // 从fromChannel的position位置开始读取,读取count到toChannel中
    fromChannel.transferTo(position, count, toChannel);
}
```

#### 5.1.3 聚集和分散

*   聚集（gather）：写入Channel是指在写操作时将多个buffer的数据写入同一个Channel，因此，Channel 将多个Buffer中的数据“聚集（gather）”后发送到Channel。
    
*   分散（scatter）：从Channel中读取是指在读操作时将读取的数据写入多个buffer中。因此，Channel将从Channel中读取的数据“分散（scatter）”到多个Buffer中。
    

scatter / gather经常用于需要将传输的数据分开处理的场合，例如传输一个由消息头和消息体组成的消息，你可能会将消息体和消息头分散到不同的buffer中，这样你可以方便的处理消息头和消息体。

*   `long write(ByteBuffer[] srcs)` 将ByteBuffer\[\]到中的数据全部写入（聚集）到 Channel
*   `long read(ByteBuffer[] dsts)` 将Channel到中的数据读取出来，然后全部写入（分散）到ByteBuffer\[\]

* * *

*   Gathering Writes（聚集）：是指数据从多个buffer写入到同一个channel。如下图描述：

![在这里插入图片描述](https://img-blog.csdnimg.cn/133d498c8aa1469faf78e396fc427282.png#pic_center)

*   Scattering Reads（分散）：是指数据从一个channel读取到多个buffer中。如下图描述：

![在这里插入图片描述](https://img-blog.csdnimg.cn/b12a0f0953a34da78cdfb5158291bed7.png#pic_center)

*   测试代码：

```java
package com.dfbz.channel;

import org.junit.Test;

import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @author lscl
 * @version 1.0
 * @intro:
 */
public class Demo03_聚集和分散 {

    // 聚集
    @Test
    public void gather() throws IOException {
        RandomAccessFile raf = new RandomAccessFile("004.txt", "rw");
        FileChannel channel = raf.getChannel();

        ByteBuffer header = ByteBuffer.wrap("{'content-Type':'application/json'}".getBytes());
        ByteBuffer body = ByteBuffer.wrap("{'username':'admin','password':'123'}".getBytes());

        ByteBuffer[] bufferArray = {header, body};

        // 将多个Buffer聚集到一起写入到channel中
        channel.write(bufferArray);

        channel.close();
        raf.close();
    }


    // 分散
    @Test
    public void test2() throws IOException {
        RandomAccessFile raf = new RandomAccessFile("004.txt", "rw");
        FileChannel channel = raf.getChannel();

        ByteBuffer header = ByteBuffer.allocate("{'content-Type':'application/json'}".getBytes().length);
        ByteBuffer body = ByteBuffer.allocate("{'username':'admin','password':'123'}".getBytes().length);

        ByteBuffer[] bufferArray = {header, body};

        // 将channel中的数据分散读取,然后逐个写入到每个Buffer中
        channel.read(bufferArray);

        for (ByteBuffer buffer : bufferArray) {

            System.out.println(new String(buffer.array(),0,buffer.array().length));
        }
        channel.close();
        raf.close();
    }
}
```

### 5.2 DatagramChannel

#### 5.2.1 DatagramChannel简介

DatagramChannel是用于UDP编程的Channel；获取到了DatagramChannel之后，可以使用Channel直接发送Buffer数据；因为UDP是无连接的网络协议，因此使用DatagramChannel发送的Buffer数据在发送时都会被封装成UDP报文，并且存在UDP协议的特性；

*   **发送和接收报文**：

在使用DatagramChannel发送数据时，必须通过`InetSocketAddress`类来指定接收端的地址和端口；而在接收数据时，接收端的Channel必须先通过`InetSocketAddress`类绑定一个地址和端口；

*   **读取和写入数据**：

DatagramChannel不仅可以发送报文和接收报文，还可以读取DatagreamChannel中的数据，或往DatagreamChannel中写入数据；与发送和接收不同，在使用DatagramChannel发送或接收时，DatagramChannel充当一个接收器/发送器的角色，**自己本身并不存储那些数据**；而是将数据接收到一个Buffer中，而在使用DatagramChannel读取/写入数据时，数据可以从Buffer中读取到Channel中，也可以从Channel写出到Buffer；

#### 5.2.2 DatagramChannel的获取

在Java NIO中，我们可以通过DatagreamChannel来直接打开一个Channel，也可以DatagreamSocket可以获取到一个属于该Socket的Channel，**但该Socket必须是由Channel获取的Socket**；这两种方式获取到的Channel是同一个；

*   DatagreamChannel方法：
    
    *   `public static DatagramChannel open()` ：打开一个基于UDP协议的Channel管道；
    *   `public DatagramSocket socket()`：通过Channel获取一个Socket；
*   DatagreamSocket方法：
    
    *   `public DatagramChannel getChannel()`：获取该Socket的Channel管道；

测试代码：

```java
package com.dfbz.channel.datagramChannel;

import org.junit.Test;

import java.net.DatagramSocket;
import java.nio.channels.DatagramChannel;

/**
 * @author lscl
 * @version 1.0
 * @intro:
 */
public class Demo01_DatagramChannel的获取 {

    @Test
    public void test() throws Exception {
        // 通过DatagramChannel打开一个Channel
        DatagramChannel channel = DatagramChannel.open();

        // 通过channel也可以获取一个socket
        DatagramSocket socket = channel.socket();

        // 通过socket也可以获取Channel
        DatagramChannel channel2 = socket.getChannel();
        System.out.println(channel2.getClass());            // class sun.nio.ch.DatagramChannelImpl

        System.out.println(channel == channel2);            // true
    }

    @Test
    public void test2() throws Exception {
        DatagramSocket socket = new DatagramSocket(9999);

        // 不能通过DatagramSocket来获取channel
        DatagramChannel channel = socket.getChannel();

        System.out.println(channel);            // null
    }
}
```

#### 5.2.3 DatagramChannel的常用方法

*   `public DatagramChannel bind(SocketAddress local)`：绑定当前客户端的地址和端口，其他Channel向当前Channel发送数据时指定的地址。**在接收UDP报文时，必须先绑定；**
*   `public SocketAddress receive(ByteBuffer dst)`：接收UDP报文，将接收到的UDP报文赋值给dst；
*   `public int send(ByteBuffer src, SocketAddress target)`：发送一个UDP报文；并指定报文要发送的地址和端口
*   `public DatagramChannel connect(SocketAddress remote)`：用于连接其他的DatagramChannel ，DatagramChannel之间建立连接后可以相互读取/写入数据；

> Tips：由于UDP是无连接的，使用connect方法连接到特定地址并不会像TCP通道那样创建一个真正的连接。而是锁住DatagramChannel ，让其只能从特定地址收发数据。因此即使是连接的地址不存在，也不会报错；

##### 1）发送和接收

测试代码：

```java
package com.dfbz.channel.datagramChannel;

import org.junit.Test;

import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.DatagramChannel;

/**
 * @author lscl
 * @version 1.0
 * @intro:
 */
public class Demo02_发送和接收 {

    /**
     * 发送端
     *
     * @throws Exception
     */
    @Test
    public void sender() throws Exception {

        // 通过DatagramChannel打开一个Channel
        DatagramChannel channel = DatagramChannel.open();

        // 往该Channel写入数据
        channel.send(ByteBuffer.wrap("hello".getBytes()), new InetSocketAddress("127.0.0.1", 9999));

        channel.close();
    }

    /**
     * 接收端
     *
     * @throws Exception
     */
    @Test
    public void receive() throws Exception {

        // 通过DatagramChannel打开一个Channel
        DatagramChannel channel = DatagramChannel.open();

        // 绑定地址(用于接收该地址发送过来的数据)
        channel.bind(new InetSocketAddress("127.0.0.1", 9999));

        ByteBuffer buffer = ByteBuffer.allocate(1024);

        // 使用Buffer接收报文
        channel.receive(buffer);

        System.out.println(new String(buffer.array(), 0, buffer.array().length));
    }
}
```

##### 2）读取和写入

![在这里插入图片描述](https://img-blog.csdnimg.cn/b1bb0ecd7efd40d595b4876248c99ec3.png#pic_center)

*   读取端：

```java
package com.dfbz.channel.datagramChannel;

import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.DatagramChannel;

/**
 * @author lscl
 * @version 1.0
 * @intro:
 */
public class Demo03_读取端 {


    public static void main(String[] args) throws Exception {

        // 获取Channel
        DatagramChannel channel = DatagramChannel.open();

        /*
            绑定一个地址
                1) 用于接收该地址发送的UDP报文
                2) 用于其他DatagramChannel与当前Channel建立连接(逻辑连接),待会可以使用Channel从127.0.0.1的9999端口读取数据
         */
        channel.bind(new InetSocketAddress("127.0.0.1", 9999));

        /*
            连接一个地址:
                1) 建立一个UDP逻辑连接,如果需要读取127.0.0.2主机的数据则必须建立逻辑连接
                2) 建立好逻辑连接后,可以使用channel向127.0.0.2主机写入数据
         */
        channel.connect(new InetSocketAddress("127.0.0.2", 9999));

        System.out.println("开始接收数据: ");
        System.out.println("----------------");

        // 创建Buffer,用读取Channel中的数据
        ByteBuffer readBuffer = ByteBuffer.allocate(1024);                  // position=0,limit=1024,capacity=1024

        while (true) {

            // 将数据从Channel中读取出来,写入到readBuffer中
            channel.read(readBuffer);

            // 预备下次一次从Channel读出数据写入到Buffer中[position=0,limit=capacity]
            readBuffer.clear();

            System.out.println("接收到来自【" + channel.getRemoteAddress() + "】的数据: " + new String(readBuffer.array(), 0, readBuffer.array().length));
        }
    }
}
```

*   写入端：

```java
package com.dfbz.channel.datagramChannel;

import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.DatagramChannel;
import java.util.Scanner;

/**
 * @author lscl
 * @version 1.0
 * @intro:
 */
public class Demo04_写入端 {
    public static void main(String[] args) throws Exception {

        // 获取Channel
        DatagramChannel channel = DatagramChannel.open();

        /*
            绑定一个地址
                1) 用于接收该地址发送的UDP报文
                2) 用于其他DatagramChannel与当前Channel建立连接(逻辑连接),待会可以使用Channel从127.0.0.2的9999端口读取数据
         */
        channel.bind(new InetSocketAddress("127.0.0.2", 9999));

        // 连接一个地址: 与指定的地址建立逻辑连接,用于向这个地址发送数据,待会可以使用Channel向写入数据到127.0.0.1的9999端口
        channel.connect(new InetSocketAddress("127.0.0.1", 9999));

        // 创建Buffer
        ByteBuffer writeBuffer = ByteBuffer.allocate(1024);       // position=0,limit=1024,capacity=1024

        // 获取一个扫描器
        Scanner scanner = new Scanner(System.in);

        while (true) {

            System.out.println("请输入数据: ");

            // 接收控制台输入的数据
            String str = scanner.nextLine();

            // 将控制台输入的数据添加到buffer中
            writeBuffer.put(str.getBytes());                    // buffer的position会进行位移

            // limit=position,position=0
            writeBuffer.flip();

            // 从Buffer中读取数据出来,往Channel中写入数据          // buffer的position会进行位移
            channel.write(writeBuffer);

            // position=0,limit=capacity(预备下一次写入)
            writeBuffer.clear();

            System.out.println("使用Channel向【" + channel.getRemoteAddress() + "】发送了数据: " + str);
        }
    }
}
```

使用一个Channel进行读取和写入：

```java
package com.dfbz.channel.datagramChannel;

import org.junit.Test;

import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.DatagramChannel;

/**
 * @author lscl
 * @version 1.0
 * @intro:
 */
public class Demo05_读取和写入 {

    /**
     * 读取/写入数据
     *
     * @throws Exception
     */
    @Test
    public void readerAndWriter() throws Exception {

        // 获取Channel
        DatagramChannel channel = DatagramChannel.open();

        /*
            绑定一个端口(本机)
                1) 接收127.0.0.1主机9999端口发送的UDP报文
                2) 用于其他DatagramChannel与当前Channel建立连接(逻辑连接),待会可以使用Channel从127.0.0.1的9999端口读取数据
         */
        channel.bind(new InetSocketAddress("127.0.0.1", 9999));

        /*
            连接一个地址:
                1) 建立一个UDP逻辑连接,如果需要读取127.0.0.2主机的数据则必须建立逻辑连接
                2) 建立好逻辑连接后,可以使用channel向127.0.0.2主机写入数据
         */
        channel.connect(new InetSocketAddress("127.0.0.1", 9999));

        // 创建Buffer,用于往Channel写入数据
        ByteBuffer writeBuffer = ByteBuffer.wrap("hello".getBytes());       // position=0,limit=5,capacity=5

        // 创建Buffer,用读取Channel中的数据
        ByteBuffer readBuffer = ByteBuffer.allocate(1024);                  // position=0,limit=1024,capacity=1024

        while (true) {

            // 将数据从writeBuffer中读取出来,写入到Channel中
            channel.write(writeBuffer);                                     // writeBuffer:[position=5,limit=5,capacity=5]

            // 切换写模式读模式,writeBuffer:[position=0,limit=5,capacity=5]
            writeBuffer.flip();

            // 将数据从Channel中读取出来,写入到readBuffer中
            channel.read(readBuffer);                                       // readBuffer:[position=5,limit=1024,capacity=1024]

            // 切换写模式,readBuffer:[position=0,limit=1024,capacity=1024]
            readBuffer.clear();

            System.out.println("Channel的数据: " + new String(readBuffer.array(), 0, readBuffer.array().length));

            Thread.sleep(1000);
        }
    }
}
```

### 5.3 SocketChannel与ServerSocketChannel

#### 5.3.1 Channel的简介

BIO中Socket编程的两个核心类分别为：Socket（代表客户端）和ServerSocket（代表服务器端），通过ServerSocket的accept可以接收一个客户端的Socket；

在NIO中，提供有SocketChannel和ServerSocketChannel，分别代表客户端和服务端；底层依旧采用TCP协议进行数据的网络传输，同时这些Channel还支持非阻塞方式运行，这一点与原生的Socket/ServerSocket有很大的不同；例如ServerSocketChannel在接收一个客户端时，如果还未有客户端来连接服务端，那么accept会返回null，而不是将当前线程阻塞；

#### 5.3.2 Channel的获取

通过ServerSocketChannel也可以来获取一个SocketChannel；也可以和DatagramChannel一样，通过open方法来打开一个管道；并且通过Socket可以获取SocketChannel，通过ServerSocket可以获取ServerSocketChannel；

* * *

和DatagramChannel一样，虽然通过Socket可以获取Channel，但该Socket必须是由Channel获取的Socket；因为原生的Socket的getChannel()方法永远返回的是null；

![在这里插入图片描述](https://img-blog.csdnimg.cn/386627fa206c448bb017c6fb27303c71.png#pic_center)

*   测试代码：

```java
package com.dfbz.channel.socketChannel;

import org.junit.Test;

import java.net.ServerSocket;
import java.net.Socket;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

/**
 * @author lscl
 * @version 1.0
 * @intro:
 */
public class Demo01_Channel的获取 {

    @Test
    public void test1() throws Exception {
        // 获取socketChannel
        SocketChannel socketChannel = SocketChannel.open();
        // 通过channel获取socket
        Socket socket = socketChannel.socket();

        // 通过socket也可以获取对应的Channel
        System.out.println(socket.getChannel() == socketChannel);           // true
        System.out.println(socket.getClass());              // class sun.nio.ch.SocketAdaptor(并不是一个原生的Socket对象)

        // serverSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        // 通过channel获取socket
        ServerSocket serverSocket = serverSocketChannel.socket();

        // 通过socket也可以获取对应的Channel
        System.out.println(serverSocket.getChannel() == serverSocketChannel);           // true
        System.out.println(serverSocket.getClass());        // class sun.nio.ch.ServerSocketAdaptor(并不是一个原生的ServerSocket对象)
    }


    @Test
    public void test2() throws Exception {

        // 原生的socket并不能获取Channel
        Socket socket = new Socket();
        SocketChannel socketChannel = socket.getChannel();
        System.out.println(socketChannel);              // null

        // 原生的ServerSocket并不能获取Channel
        ServerSocket serverSocket = new ServerSocket();
        ServerSocketChannel serverSocketChannel = serverSocket.getChannel();
        System.out.println(serverSocketChannel);        // null
    }


    @Test
    public void test3() throws Exception {

        // serverSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        // 接收到一个socketChannel(客户端)
        SocketChannel socketChannel = serverSocketChannel.accept();
    }
}
```

#### 5.3.3 Channel的常用方法

*   ServerSocketChannel：
    *   `public static ServerSocketChannel open()`：获取一个ServerSocketChannel
    *   `public ServerSocket socket()`：通过当前Channel获取Socket
    *   `public ServerSocketChannel bind(SocketAddress local)`：绑定一个地址，用于客户端（SocketChannel）来连接
    *   `public SocketChannel accept()`：接收一个客户端（SocketChannel）；默认情况下，如果没有客户端来连接，那么accept会使得当前线程一直处于等待状态；
    *   `public SelectableChannel configureBlocking(boolean block)`：将当前Channel设置为非阻塞模式；默认为true（阻塞模式）
    *   `public boolean isBlocking()`：判断当前Channel是否为非阻塞模式；默认true（阻塞模式）
*   SocketChannel：
    *   `public static SocketChannel open()` ：获取一个SocketChannel
    *   `public Socket socket()`：通过当前Channel获取Socket；
    *   `public SocketChannel bind(SocketAddress local)`：绑定一个地址，默认是本机地址，SocketChannel的该方法没有意义，因为SocketChannel是一个客户端，用于连接ServerSocketChannel，通过ServerSocketChannel可以获取客户端的地址，默认情况下SocketChannel为本机地址，并随机分配一个端口；
    *   `public boolean connect(SocketAddress remote)`：用于连接服务端
    *   `public int read(ByteBuffer dst)`：读取Channel中的数据到Buffer中
    *   `public int write(ByteBuffer src)`：将Buffer中的数据写入到Channel中；
    *   `public SelectableChannel configureBlocking(boolean block)`：将当前Channel设置为非阻塞模式；默认为true（阻塞模式）
    *   `public boolean isBlocking()`：判断当前Channel是否为非阻塞模式；默认true（阻塞模式）

##### 1）基本读写

*   服务端：

```java
package com.dfbz.channel.socketChannel和serverSocketChannel;

import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

/**
 * @author lscl
 * @version 1.0
 * @intro:
 */
public class Demo02_基本演示_服务端 {
    public static void main(String[] args) throws Exception{
        // 创建一个服务器
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        System.out.println("等待客户端连接....");

        // 绑定一个地址
        serverSocketChannel.bind(new InetSocketAddress("127.0.0.1", 9999));

        /*
            接收一个客户端(如果没有客户端来连接则会造成阻塞)
            接收到了与客户端交互的Channel后,通过SocketChannel既可以向客户端写出数据,又可以读取来自客户端发送的数据
         */
        SocketChannel socketChannel = serverSocketChannel.accept();

        System.out.println("客户端【" + socketChannel.getRemoteAddress() + "】连接成功成功....");

        // 准备一个Buffer
        ByteBuffer buffer = ByteBuffer.wrap("hello".getBytes());

        // 往客户端写入数据
        socketChannel.write(buffer);

        socketChannel.close();
        serverSocketChannel.close();
    }
}
```

*   客户端：

```java
package com.dfbz.channel.socketChannel和serverSocketChannel;

import org.junit.Test;

import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;

/**
 * @author lscl
 * @version 1.0
 * @intro:
 */
public class Demo03_基本演示_客户端 {
    @Test
    public void server() throws Exception {
        SocketChannel socketChannel = SocketChannel.open();
        // 连接到一个地址(如果该地址不存在,则默认情况下会阻塞,等到一定时间后抛出异常)
        socketChannel.connect(new InetSocketAddress("127.0.0.1", 9999));

        // 准备一个Buffer用于接收服务端的数据
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        // 读取服务端的数据,返回数据的长度
        int len = socketChannel.read(buffer);

        System.out.println("数据长度: " + len);
        System.out.println(new String(buffer.array(), 0, buffer.array().length));
    }
}
```

##### 2）拷贝图片案例

##### 3）在线聊天案例

 

仅限学习、交流使用！

![](https://g.csdnimg.cn/extension-box/1.1.6/image/qq.png) QQ群名片

![](https://g.csdnimg.cn/extension-box/1.1.6/image/ic_move.png)

本文转自 <https://blog.csdn.net/Bb15070047748/article/details/125438312>，如有侵权，请联系删除。