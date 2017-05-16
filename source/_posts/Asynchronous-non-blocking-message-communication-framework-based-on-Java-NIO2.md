title: 基于Java NIO2实现的异步非阻塞消息通信框架
date: 2016-02-26 22:11:30
tags: [Java]
categories: Programming Notes

---
#### 前奏
因为`NIO`并不容易掌握，所以这注定会是一篇长文，而且即便篇幅很大，亦难以把很多细节解释清楚，只能侧重于从整体上进行把握，并实现一个简单的客户端服务端消息通信框架作为例子，以便有需要的开发人员参考之。借用淘宝伯岩给出的忠告就是
- 尽量不要尝试实现自己的`NIO`框架，除非有经验丰富的工程师
- 尽量使用经过广泛实践的开源`NIO`框架`Mina/Netty/xSocket`
- 尽量使用最新版稳定版`JDK`
- 遇到问题的时候，可以先看下`Java`的`Bug Database`

`Asynchronous I/O`是在`JDK7`中提出的异步非阻塞`I/O`，习惯上称之为`NIO2`，也叫`AIO`，`AIO`是对`JDK1.4`中提出的同步非阻塞`I/O`的进一步增强，主要包括
- 更新的`Path`类，该类在`NIO`里对文件系统进行了进一步的抽象，用来替换原来的`java.io.File`，可以通过`File.toPath()`和`Path.toFile()`将`File`和`Path`进行相互转换
- `File Attributes`，`java.nio.file.attribute`针对文件属性提供了各种用户所需的元数据，不同操作系统使用的类不太一样，支持的属性分类有
    - BasicFileAttributeView
    - DosFileAttributeView
    - PosixFileAttributeView
    - FileOwnerAttributeView
    - AclFileAttributeView
    - UserDefinedFileAttributeView
- `Symbolic and Hard Links`，相当于用`Java`程序实现`Linux`中的`ln`命令
- `Watch Service API`，作为一个线程安全的服务用于监控对象的变化和事件，以前直接用`Java`监控文件系统的变化是不可能的，只能通过`JNI`的方式调用操作系统的`API`，而在`JDK7`中这部分被加入到了标准库里
- `Random Access Files`主要提供了一个`SeekableByteChannel`接口，配合`ByteBuffer`使得随机访问文件更加方便
- `Sockets API`主要是`NIO1`中的`Selector`模式实现同步非阻塞
- `Asynchronous Channel API`由`NIO1`中的`Selector`模式变成方法回调模式，使用更加方便，主要是可以异步实现文件的读写了

#### AIO应用开发
##### Future方式
`Future`是在`JDK1.5`中加入`Java`并发包的，该接口提供`get()`方法用于获取任务完成之后的处理结果。在`AIO`中，可以接受一个`I/O`连接请求，返回一个`Future`对象，然后可以基于该返回对象进行后续的操作，包括使其阻塞、查看是否完成、超时异常，使用方式如下。
**服务端代码**
```java
import lombok.extern.log4j.Log4j2;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.StandardSocketOptions;
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.charset.Charset;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;

/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/24 16:57
 * </p>
 * <p>
 * ClassName:ServerOnFuture
 * </p>
 * <p>
 * Description:基于Future的NIO2服务端实现，此时的服务端还无法实现多客户端并发，如果有多个客户端并发连接该服务端的话，
 * 客户端会出现阻塞，待前一个客户端处理完毕，服务端才会接受下一个客户端的连接并处理
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
@Log4j2
public class ServerOnFuture {
    static final int DEFAULT_PORT = 7777;
    static final String IP = "127.0.0.1";
    static ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    public static void main(String[] args) {
        try (AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel.open()) {
            if (serverSocketChannel.isOpen()) {
                serverSocketChannel.setOption(StandardSocketOptions.SO_REUSEADDR, true);
                serverSocketChannel.bind(new InetSocketAddress(IP, DEFAULT_PORT));
                log.info("Waiting for connections...");
                while (true) {
                    Future<AsynchronousSocketChannel> channelFuture = serverSocketChannel.accept();
                    try (AsynchronousSocketChannel socketChannel = channelFuture.get()) {
                        log.info("Incoming connection from : " + socketChannel.getRemoteAddress());
                        while (socketChannel.read(buffer).get() != -1) {
                            buffer.flip();
                            // Java NIO2或者Java AIO报： java.util.concurrent.ExecutionException: java.io.IOException: 指定的网络名不再可用。
                            // 此处要注意，千万不能直接操作buffer，否则客户端会阻塞并报错，“java.util.concurrent.ExecutionException: java.io.IOException: 指定的网络名不再可用。”
                            ByteBuffer duplicate = buffer.duplicate();
                            showMessage(duplicate);
                            socketChannel.write(buffer).get();
                            if (buffer.hasRemaining()) {
                                buffer.compact();
                            } else {
                                buffer.clear();
                            }
                        }
                        log.info(socketChannel.getRemoteAddress() + " was successfully served!");
                    } catch (InterruptedException | ExecutionException e) {
                        log.error(e);
                    }
                }
            } else {
                log.warn("The asynchronous server-socket channel cannot be opened!");
            }

        } catch (IOException e) {
            log.error(e);
        }
    }

    protected static void showMessage(ByteBuffer buffer) {
        CharBuffer decode = Charset.defaultCharset().decode(buffer);
        log.info(decode.toString());
    }
}
```

**客户端代码**
```java
import lombok.extern.log4j.Log4j2;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.StandardSocketOptions;
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.charset.Charset;
import java.util.Random;
import java.util.concurrent.ExecutionException;

/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/24 17:48
 * </p>
 * <p>
 * ClassName:ClientOnFuture
 * </p>
 * <p>
 * Description:基于Future的NIO2客户端实现
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
@Log4j2
public class ClientOnFuture {
    static final int DEFAULT_PORT = 7777;
    static final String IP = "127.0.0.1";
    static ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    public static void main(String[] args) {
        try (AsynchronousSocketChannel socketChannel = AsynchronousSocketChannel.open()) {
            if (socketChannel.isOpen()) {
                //设置一些选项，非必选项，可使用默认设置
                socketChannel.setOption(StandardSocketOptions.SO_RCVBUF, 128 * 1024);
                socketChannel.setOption(StandardSocketOptions.SO_SNDBUF, 128 * 1024);
                socketChannel.setOption(StandardSocketOptions.SO_KEEPALIVE, true);
                Void aVoid = socketChannel.connect(new InetSocketAddress(IP, DEFAULT_PORT)).get();
                //返回null表示连接成功
                if (aVoid == null) {
                    Integer messageLength = socketChannel.write(ByteBuffer.wrap("Hello Server！".getBytes())).get();
                    log.info(messageLength);
                    while (socketChannel.read(buffer).get() != -1) {
                        buffer.flip();//写入buffer之后，翻转，之后可以从buffer读取，或者将buffer内容写入通道
                        CharBuffer decode = Charset.defaultCharset().decode(buffer);
                        log.info(decode.toString());
                        if (buffer.hasRemaining()) {
                            buffer.compact();
                        } else {
                            buffer.clear();
                        }
                        int r = new Random().nextInt(1000);
                        if (r == 50) {
                            log.info("Client closed!");
                            break;
                        } else {
                            // Java NIO2或者Java AIO报： Exception in thread "main" java.nio.channels.WritePendingException
                            // 此处注意，如果在频繁调用write()的时候，在上一个操作没有写完的情况下，调用write会触发WritePendingException异常，
                            // 所以此处最好在调用write()之后调用get()以便明确等到有返回结果
                            socketChannel.write(ByteBuffer.wrap("Random number : ".concat(String.valueOf(r)).getBytes())).get();
                        }

                    }
                } else {
                    log.warn("The connection cannot be established!");
                }
            } else {
                log.warn("The asynchronous socket-channel cannot be opened!");
            }
        } catch (IOException | InterruptedException | ExecutionException e) {
            log.error(e);
        }
    }
}
```

##### Future方式实现为多客户端并发服务
如何让服务端同时可以接受多个客户端的连接呢？一个简单的处理方法就是使用`ExecutorService`。每次新建一个连接，并且获得返回值之后，这个返回值就是一个`AsynchronousSocketChannel`的通道，将其提交给线程池，由一个工作线程进行后续处理。然后一个新的线程准备好在等待接受下一个连接。代码示例如下。
```java
import lombok.extern.log4j.Log4j2;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.StandardSocketOptions;
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.charset.Charset;
import java.util.concurrent.*;

/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/24 17:38
 * </p>
 * <p>
 * ClassName:ServerOnFutureForMultiClients
 * </p>
 * <p>
 * Description:基于Future实现的可以接受多客户端并发的Java NIO2服务端实现
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
@Log4j2
public class ServerOnFutureForMultiClients {
    static final int DEFAULT_PORT = 7777;
    static final String IP = "127.0.0.1";
    static ExecutorService taskExecutorService = Executors.newCachedThreadPool(Executors.defaultThreadFactory());
    static ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    public static void main(String[] args) {
        try (AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel.open()) {
            if (serverSocketChannel.isOpen()) {
                serverSocketChannel.setOption(StandardSocketOptions.SO_REUSEADDR, true);
                serverSocketChannel.bind(new InetSocketAddress(IP, DEFAULT_PORT));
                log.info("Waiting for connections...");
                while (true) {
                    Future<AsynchronousSocketChannel> socketChannelFuture = serverSocketChannel.accept();
                    try {
                        final AsynchronousSocketChannel socketChannel = socketChannelFuture.get();
                        Callable<String> worker = new Callable<String>() {
                            @Override
                            public String call() throws Exception {
                                String s = socketChannel.getRemoteAddress().toString();
                                log.info("Incoming connection from : " + s);
                                while (socketChannel.read(buffer).get() != -1) {
                                    buffer.flip();
                                    ByteBuffer duplicate = buffer.duplicate();
                                    showMessage(duplicate);
                                    socketChannel.write(buffer).get();
                                    if (buffer.hasRemaining()) {
                                        buffer.compact();
                                    } else {
                                        buffer.clear();
                                    }
                                }
                                socketChannel.close();
                                log.info(s + " was successfully served!");
                                return s;
                            }
                        };
                        taskExecutorService.submit(worker);
                    } catch (InterruptedException | ExecutionException e) {
                        log.error(e);
                        taskExecutorService.shutdown();
                        while (!taskExecutorService.isTerminated()) {

                        }
                        break;
                    }
                }
            } else {
                log.warn("The asynchronous server-socket channel cannot be opened!");
            }
        } catch (IOException e) {
            log.error(e);
        }
    }

    protected static void showMessage(ByteBuffer buffer) {
        CharBuffer decode = Charset.defaultCharset().decode(buffer);
        log.info(decode.toString());
    }
}
```

##### Callback方式
方法回调模式，即提交一个`I/O`操作请求，并且指定一个`CompletionHandler`。当异步操作完成时，便会发一个通知，此时该`CompletionHandler`对象覆写的方法将被调用，如果成功调用`completed`方法，如果失败调用`failed`方法，首先看下`Java API`。
```java
public interface CompletionHandler<V,A> {

    /**
     * Invoked when an operation has completed.
     *
     * @param   result 操作结果
     *          The result of the I/O operation.
     * @param   attachment 提交请求时的参数，通常会封装一个连接环境
     *          The object attached to the I/O operation when it was initiated.
     */
    void completed(V result, A attachment);

    /**
     * Invoked when an operation fails.
     *
     * @param   exc
     *          The exception to indicate why the I/O operation failed
     * @param   attachment
     *          The object attached to the I/O operation when it was initiated.
     */
    void failed(Throwable exc, A attachment);
}
```
`AIO`提供了四种类型的异步通道以及不同的`I/O`操作可以接收一个`CompletionHandler`对象，分别是：
- AsynchronousSocketChannel：connect，read，write
- AsynchronousFileChannel：lock，read，write
- AsynchronousServerSocketChannel：accept
- AsynchronousDatagramChannel：read，write，send，receive

**服务端示例代码如下**
```java
import lombok.extern.log4j.Log4j2;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.StandardSocketOptions;
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.nio.charset.Charset;
import java.util.concurrent.ExecutionException;

/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/24 20:14
 * </p>
 * <p>
 * ClassName:ServerOnCompletionHandler
 * </p>
 * <p>
 * Description:基于CompletionHandler实现NIO2的服务端
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
@Log4j2
public class ServerOnCompletionHandler {
    static final int DEFAULT_PORT = 7777;
    static final String IP = "127.0.0.1";

    public static void main(String[] args) {
        try (final AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel.open()) {
            if (serverSocketChannel.isOpen()) {
                serverSocketChannel.setOption(StandardSocketOptions.SO_RCVBUF, 4 * 1024);
                serverSocketChannel.setOption(StandardSocketOptions.SO_REUSEADDR, true);
                serverSocketChannel.bind(new InetSocketAddress(IP, DEFAULT_PORT));
                log.info("Waiting for connections...");
                serverSocketChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
                    final ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

                    @Override
                    public void completed(AsynchronousSocketChannel socketChannel, Void attachment) {
                        //注意接收一个连接之后，紧接着可以接收下一个连接，所以必须再次调用accept方法
                        serverSocketChannel.accept(null, this);
                        try {
                            log.info("Incoming connection from : " + socketChannel.getRemoteAddress());
                            while (socketChannel.read(buffer).get() != -1) {
                                buffer.flip();
                                final ByteBuffer duplicate = buffer.duplicate();
                                final CharBuffer decode = Charset.defaultCharset().decode(duplicate);
                                log.info(decode.toString());
                                socketChannel.write(buffer).get();
                                if (buffer.hasRemaining()) {
                                    buffer.compact();
                                } else {
                                    buffer.clear();
                                }
                            }

                        } catch (InterruptedException | ExecutionException | IOException e) {
                            log.error(e);
                        } finally {
                            try {
                                socketChannel.close();
                            } catch (IOException e) {
                                log.error(e);
                            }

                        }
                    }

                    @Override
                    public void failed(Throwable exc, Void attachment) {
                        serverSocketChannel.accept(null, this);
                        throw new UnsupportedOperationException("Cannot accept connections!");
                    }
                });
                //主要是阻塞作用，因为AIO是异步的，所以此处不阻塞的话，主线程很快执行完毕，并会关闭通道
                System.in.read();
            } else {
                log.warn("The asynchronous server-socket channel cannot be opened!");
            }
        } catch (IOException e) {
            log.error(e);
        }
    }
}
```
**客户端代码如下**
```java
import lombok.extern.log4j.Log4j2;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.StandardSocketOptions;
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.nio.charset.Charset;
import java.util.Random;
import java.util.concurrent.ExecutionException;

/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/25 10:13
 * </p>
 * <p>
 * ClassName:ClientOnCompletionHandler
 * </p>
 * <p>
 * Description:基于匿名内部类形式的CompletionHandler客户端实现
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
@Log4j2
public class ClientOnCompletionHandler {
    static final int DEFAULT_PORT = 7777;
    static final String IP = "127.0.0.1";

    public static void main(String[] args) {
        try (final AsynchronousSocketChannel socketChannel = AsynchronousSocketChannel.open()) {
            if (socketChannel.isOpen()) {
                socketChannel.setOption(StandardSocketOptions.SO_RCVBUF, 128 * 1024);
                socketChannel.setOption(StandardSocketOptions.SO_SNDBUF, 128 * 1024);
                socketChannel.setOption(StandardSocketOptions.SO_KEEPALIVE, true);
                socketChannel.connect(new InetSocketAddress(IP, DEFAULT_PORT), null, new CompletionHandler<Void, Void>() {
                    final ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

                    @Override
                    public void completed(Void result, Void attachment) {
                        try {
                            log.info("Successfully connected at : " + socketChannel.getRemoteAddress());
                            socketChannel.write(ByteBuffer.wrap("Hello Server！".getBytes())).get();
                            while (socketChannel.read(buffer).get() != -1) {
                                buffer.flip();
                                ByteBuffer duplicate = buffer.duplicate();
                                CharBuffer decode = Charset.defaultCharset().decode(duplicate);
                                log.info(decode.toString());
//                                只要还有多余位置就可以继续从通道读入buffer，但是其实没必要，除非你要保留上一次通信的信息，一般全清空即可
//                                if (buffer.hasRemaining()) {
//                                    buffer.compact();
//                                } else {
                                buffer.clear();
//                                }
                                int r = new Random().nextInt(1000);
                                if (r == 50) {
                                    log.warn("Client closed!");
                                    break;
                                } else {
                                    socketChannel.write(ByteBuffer.wrap("Random number ".concat(String.valueOf(r)).getBytes())).get();
                                }
                            }
                        } catch (IOException | InterruptedException | ExecutionException e) {
                            e.printStackTrace();
                        } finally {
                            try {
                                socketChannel.close();
                            } catch (IOException e) {
                                log.error(e);
                            }

                        }
                    }

                    @Override
                    public void failed(Throwable exc, Void attachment) {
                        log.error("Connection cannot be established!");
                        throw new UnsupportedOperationException("Connection cannot be established!");
                    }
                });
                //如果没有可读取的数据，那么返回-1，该方法阻塞直到有可读取数据
                System.in.read();
            } else {
                log.warn("The asynchronous socket channel cannot be opened!");
            }
        } catch (IOException e) {
            log.error(e);
        }


    }
}
```
##### Reader/Writer方式实现
其实除了使用匿名内部类的形式外，还有可以指定读写者的`read`和`write`方法，另外你还可以指定超时时间，这种实现方式相对来说比匿名内部类形式看起来代码解耦合更好，代码更简洁。
**抽象的接口**
```java
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;

/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/19 17:58
 * </p>
 * <p>
 * ClassName:Callback
 * </p>
 * <p>
 * Description:回调接口，顶层抽象，主要是设定两个泛型参数
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
public interface Callback extends CompletionHandler<Integer, AsynchronousSocketChannel> {
    // 某种程度上说，AIO编程其实是attachment编程
    @Override
    void failed(Throwable exc, AsynchronousSocketChannel socketChannel);

    @Override
    void completed(Integer result, AsynchronousSocketChannel socketChannel);
}
```
```java
/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/25 11:19
 * </p>
 * <p>
 * ClassName:ReaderCallback
 * </p>
 * <p>
 * Description:回调接口的下一层抽象，针对读操作
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
public interface ReaderCallback extends Callback {
}
```
```java
/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/19 18:09
 * </p>
 * <p>
 * ClassName:WriterCallback
 * </p>
 * <p>
 * Description:回调接口的下一层抽象，负责写操作
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
public interface WriterCallback extends Callback {
}
```
**读者**
```java
import lombok.NoArgsConstructor;
import lombok.extern.log4j.Log4j2;

import java.io.IOException;
import java.net.SocketAddress;
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.charset.Charset;

/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/16 14:05
 * </p>
 * <p>
 * ClassName:Reader
 * </p>
 * <p>
 * Description:负责服务端的读消息
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
@Log4j2
@NoArgsConstructor
public class Reader implements ReaderCallback {
    private ByteBuffer byteBuffer;

    public Reader(ByteBuffer byteBuffer) {
        log.info("An reader has been created!");
        this.byteBuffer = byteBuffer;
    }

    @Override
    public void completed(Integer result, AsynchronousSocketChannel socketChannel) {
        log.info(String.format("Reader name : %s ", Thread.currentThread().getName()));
        byteBuffer.flip();
        log.info("Message size : " + result);
        if (result != null && result < 0) {
            try {
                socketChannel.close();
            } catch (IOException e) {
                log.error(e);
            }
            return;
        }
        try {
            SocketAddress localAddress = socketChannel.getLocalAddress();
            SocketAddress remoteAddress = socketChannel.getRemoteAddress();
            log.info("localAddress : " + localAddress.toString());
            log.info("remoteAddress : " + remoteAddress.toString());
            socketChannel.write(byteBuffer, socketChannel, new Writer(byteBuffer));
        } catch (IOException e) {
            log.error(e);
        }
        ByteBuffer duplicate = byteBuffer.duplicate();
        CharBuffer decode = Charset.defaultCharset().decode(duplicate);
        log.info("Receive message from client : " + decode.toString());
    }


    @Override
    public void failed(Throwable exc, AsynchronousSocketChannel attachment) {
        log.error(exc);
        throw new RuntimeException(exc);
    }

}
```
**写者**
```java
import lombok.NoArgsConstructor;
import lombok.extern.log4j.Log4j2;

import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;

/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/16 14:05
 * </p>
 * <p>
 * ClassName:Writer
 * </p>
 * <p>
 * Description:负责服务端的写操作
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
@Log4j2
@NoArgsConstructor
public class Writer implements WriterCallback {
    private ByteBuffer byteBuffer;

    public Writer(ByteBuffer byteBuffer) {
        this.byteBuffer = byteBuffer;
        log.info("A writer has been created!");
    }

    @Override
    public void completed(Integer result, AsynchronousSocketChannel socketChannel) {
        log.debug("Message write successfully, size = " + result);
        log.info(String.format("Writer name : %s ", Thread.currentThread().getName()));
        byteBuffer.clear();
        socketChannel.read(byteBuffer, socketChannel, new Reader(byteBuffer));
    }

    @Override
    public void failed(Throwable exc, AsynchronousSocketChannel socketChannel) {
        log.error(exc);
        throw new RuntimeException(exc);
    }
}
```
**服务端代码**
```java
import lombok.extern.log4j.Log4j2;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.StandardSocketOptions;
import java.nio.channels.AsynchronousServerSocketChannel;

/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/25 10:53
 * </p>
 * <p>
 * ClassName:ServerOnReaderAndWriter
 * </p>
 * <p>
 * Description:基于读写类实现的NIO2服务端
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
@Log4j2
public class ServerOnReaderAndWriter {
    static final int DEFAULT_PORT = 7777;
    static final String IP = "127.0.0.1";

    public static void main(String[] args) {
        try (AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel.open()) {
            if (serverSocketChannel.isOpen()) {
                //服务端的通道支持两种选项SO_RCVBUF和SO_REUSEADDR，一般无需显式设置，使用其默认即可，此处仅为展示设置方法
                //在面向流的通道中，此选项表示在前一个连接处于TIME_WAIT状态时，下一个连接是否可以重用通道地址
                serverSocketChannel.setOption(StandardSocketOptions.SO_REUSEADDR, true);
                //设置通道接收的字节大小
                serverSocketChannel.setOption(StandardSocketOptions.SO_RCVBUF, 8 * 1024);
                serverSocketChannel.bind(new InetSocketAddress(IP, DEFAULT_PORT));
                log.info("Waiting for connections...");
                serverSocketChannel.accept(serverSocketChannel, new Acceptor());
                System.in.read();
            } else {
                log.warn("The connection cannot be opened!");
            }
        } catch (IOException e) {
            log.error(e);
        }
    }
}
```
**客户端代码**
```java
import lombok.extern.log4j.Log4j2;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.StandardSocketOptions;
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.charset.Charset;
import java.util.Random;
import java.util.concurrent.ExecutionException;

/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/25 15:15
 * </p>
 * <p>
 * ClassName:ClientOnReaderAndWriter
 * </p>
 * <p>
 * Description:基于读写类的NIO2客户端实现
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
@Log4j2
public class ClientOnReaderAndWriter {
    static final int DEFAULT_PORT = 7777;
    static final String IP = "127.0.0.1";
    static ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    public static void main(String[] args) {
        try (AsynchronousSocketChannel socketChannel = AsynchronousSocketChannel.open()) {
            Void aVoid = socketChannel.connect(new InetSocketAddress(IP, DEFAULT_PORT)).get();
            if (socketChannel.isOpen()) {
                if (aVoid == null) {
                    socketChannel.setOption(StandardSocketOptions.SO_RCVBUF, 128 * 1024);
                    socketChannel.setOption(StandardSocketOptions.SO_SNDBUF, 128 * 1024);
                    socketChannel.setOption(StandardSocketOptions.SO_KEEPALIVE, true);
                    socketChannel.write(ByteBuffer.wrap("Hello server".getBytes())).get();
                    while (socketChannel.read(buffer).get() != -1) {
                        buffer.flip();
                        Ch