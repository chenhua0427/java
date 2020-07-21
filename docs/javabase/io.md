##### 定义

表示主存与外部设备(硬盘、终端、网络等)传输数据的过程。

##### Java IO

Java引入了“流”的概念，表示任何由能力产生数据源或有能力接收数据源的对象。

最基本的四个抽象类：InputStream、OutputStream、Reader、Writer

最基本的方法：read()、write()

##### 编码-字节

常用编码中英文字符占字节情况

| 编码名称          | 中文 | 英文 |
| ----------------- | ---- | ---- |
| unicode、utf-16   | 4    | 4    |
| utf-8             | 3    | 1    |
| gbk、gb2312       | 2    | 1    |
| ASSII、ISO-8859-1 |      | 1    |

##### BIO(Blocking-IO)

阻塞IO，Socket的access、read、write均会阻塞线程，即一个线程只能处理一个用户请求。在高并发场景，需要为每个请求开启一个线程来满足。

##### NIO(Nonblocking-IO)

非阻塞IO，使用一种新技术解决了BIO中阻塞的问题，一个线程能同时处理多个用户请求。

原理：定义了一个集合，将每个Socket放入其中；遍历集合找出活动的socket并处理其请求。

NIO三个核心组件：Channel、Buffer、Selector。

- Channel相当于Java定义的流的概念，数据的源头/目的地，支持异步读写、缓冲。
- Buffer 数据的读/写中转地。包括：ByteBuffer、ShortBuffer、IntBuffer、LongBuffer、FloadBuffer、DoubleBuffer、CharBuffer。
- Selector是NIO的核心类，用来管理多个Channel。使用方式：1.创建Selector 2.向其注册通道 3.调用其select()方法