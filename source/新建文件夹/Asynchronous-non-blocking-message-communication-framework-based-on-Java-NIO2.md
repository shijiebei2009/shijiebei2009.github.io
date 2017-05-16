title: 基于Java NIO2实现的异步非阻塞消息通信框架
date: 2016-02-26 22:11:30
tags: [Java]
categories: Programming Notes

---
####前奏
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

####AIO应用开发
#####Future方式
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

#####Future方式实现为多客户端并发服务
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

#####Callback方式
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
#####Reader/Writer方式实现
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
                        CharBuffer decode = Charset.defaultCharset().decode(buffer);
                        log.info(decode.toString());
//                        如果调用的是clear()方法，position将被设回0，limit被设置成capacity的值。换句话说，Buffer被清空了。
//                        Buffer中的数据并未清除，只是这些标记告诉我们可以从哪里开始往Buffer里写数据。
//                        如果Buffer中有一些未读的数据，调用clear()方法，数据将“被遗忘”，意味着不再有任何标记会告诉你哪些数据被读过，哪些还没有。
//                        如果Buffer中仍有未读的数据，且后续还需要这些数据，但是此时想要先先写些数据，那么使用compact()方法。
//                        compact()方法将所有未读的数据拷贝到Buffer起始处。然后将position设到最后一个未读元素正后面。
//                        limit属性依然像clear()方法一样，设置成capacity。现在Buffer准备好写数据了，但是不会覆盖未读的数据。
                        if (buffer.hasRemaining()) {
                            buffer.compact();
                        } else {
                            buffer.clear();
                        }
                        int r = new Random().nextInt(10000);
                        if (r == 50) {
                            break;
                        } else {
                            socketChannel.write(ByteBuffer.wrap("Random number ".concat(String.valueOf(r)).getBytes())).get();
                            log.info("Client send successfully!");
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

#####Reader/Writer方式实现支持多客户端并发服务
想要使服务端支持多并发，必须要使用到`AsynchronousChannelGroup`，有关细节在下一节详述，`AsynchronousChannelGroup`用于管理异步通道资源，封装一个处理`I/O`完成的机制。该组对象关联一个线程池，可以将处理任务提交到线程池，这个组对象相当于是一个`Dispatcher`。
```java
import lombok.extern.log4j.Log4j2;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.StandardSocketOptions;
import java.nio.channels.AsynchronousChannelGroup;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

/**
 * <p>
 * Created with IntelliJ IDEA. 16/2/25 10:53
 * </p>
 * <p>
 * ClassName:ServerOnReaderAndWriterForMultiClients
 * </p>
 * <p>
 * Description:基于读写类实现的服务端，并同时支持多客户端并发
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 * WebSite: http://codepub.cn
 * Licence: Apache v2 License
 */
@Log4j2
public class ServerOnReaderAndWriterForMultiClients {
    static final int DEFAULT_PORT = 7777;
    static final String IP = "127.0.0.1";
    static AsynchronousChannelGroup threadGroup = null;
    static ExecutorService executorService = Executors.newCachedThreadPool(Executors.defaultThreadFactory());

    public static void main(String[] args) {
        try {
            threadGroup = AsynchronousChannelGroup.withCachedThreadPool(executorService, 5);
            //或者使用指定数量的线程池
            //threadGroup = AsynchronousChannelGroup.withFixedThreadPool(6, Executors.defaultThreadFactory());
        } catch (IOException e) {
            log.error(e);
        }
        try (AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel.open(threadGroup)) {
            if (serverSocketChannel.isOpen()) {
                //服务端的通道支持两种选项SO_RCVBUF和SO_REUSEADDR，一般无需显式设置，使用其默认即可，此处仅为展示设置方法
                //在面向流的通道中，此选项表示在前一个连接处于TIME_WAIT状态时，下一个连接是否可以重用通道地址
                serverSocketChannel.setOption(StandardSocketOptions.SO_REUSEADDR, true);
                //设置通道接收的字节大小
                serverSocketChannel.setOption(StandardSocketOptions.SO_RCVBUF, 8 * 1024);
                serverSocketChannel.bind(new InetSocketAddress(IP, DEFAULT_PORT));
                log.info("Waiting for connections...");
                serverSocketChannel.accept(serverSocketChannel, new Acceptor());
                threadGroup.awaitTermination(Long.MAX_VALUE, TimeUnit.SECONDS);
//                System.in.read();
            } else {
                log.warn("The connection cannot be opened!");
            }
        } catch (IOException | InterruptedException e) {
            log.error(e);
        }
    }
}
```
####线程池和Group
四种异步通道的`open`方法可以指定`group`参数，或者不指定。每个异步通道都必须关联一个组，要么是系统默认组，要么是用户创建的组。如果不使用`group`参数，`java`使用一个默认的系统范围的组对象。系统默认的组对象的线程池参数可以使用两个属性进行配置：
- java.nio.channels.DefaultThreadPool.threadFactory 默认组对象不会将其关联的线程池中的线程进行额外的配置，因此，这些线程都是`daemon`线程。
- java.nio.channels.DefaultThreadPool.initialSize: 处理`I/O`事件的最大线程数量。

组与`ExecutorService`类似，这意味着关闭过程通常是两步关闭方法。在多层次`Client`结构（例如`FTP`的控制通道需要衍生新的数据传输通道）中，如果要使用`group`，很讨厌的一点就是`group`参数传递。没有环境编程之类的工具进行辅助的话，使用者必须考虑如何有效传递`group`参数。

不使用`group`，最大的好处是不用传递`group`参数。缺点是：必须注意处理非`daemon`线程的完成和退出，不小心的话，将会导致异步通道的工作丢失；同时还需要处理线程工厂和最大线程数的配置。

####PendingException 和 AsynchronousChannel
如果一个读写操作没有完成，程序又发送一个读写操作命令，则导致`ReadPendingException`或者`WritePendingException`。如果你的程序非要这样的话，只有一个解决办法，将读写操作的命令使用队列排队进行。通常应该不会出现这种需求，如果有的话，很有可能是设计上的缺陷。

读写超时。`AsynchronousChannel`的读写操作可以指定超时参数，但是超时发生之后，传递给读写操作的`ByteBuffer`参数不应该向正常读写完成一样进行处理。通常设计如果超时发生，一般应该丢弃当前期望数据结果。

####ByteBuffer
`AIO`鼓励使用`DirectByteBuffer`。就算应用程序代码中不使用`DirectByteBuffer`，`AIO`内核实现也会使用`DirectByteBuffer`来复制外部传入的`HeadByteBuffer`内容。

`ByteBuffer`主要有两个继承的类分别是：`HeapByteBuffer`和`MappedByteBuffer`。他们的不同之处在于`HeapByteBuffer`会在`JVM`的堆上分配内存资源，而`MappedByteBuffer`的资源则会由`JVM`之外的操作系统内核来分配。`DirectByteBuffer`继承了`MappedByteBuffer`，采用了直接内存映射的方式，将文件直接映射到虚拟内存，同时减少在内核缓冲区和用户缓冲区之间的调用，尤其在处理大文件方面有很大的性能优势。但是在使用内存映射的时候会造成文件句柄一直被占用而无法删除的情况，网上也有很多介绍。

`Netty`中使用`ChannelBuffer`来处理读写，之所以废弃`ByteBuffer`，官方说法是`ChannelBuffer`简单易用并且有性能方面的优势。在`ChannelBuffer`中使用`ByteBuffer`或者`byte[]`来存储数据。同样的，`ChannelBuffer`也提供了几个标记来控制读写并以此取代`ByteBuffer`的`position`和`limit`，分别是：
`0 <= readerIndex <= writerIndex <= capacity`，同时也有类似于`mark`的`markedReaderIndex`和`markedWriterIndex`。当写入`buffer`时，`writerIndex`增加，从`buffer`中读取数据时`readerIndex`增加，而不能超过`writerIndex`。有了这两个变量后，就不用每次写入`buffer`后调用`flip()`方法，方便了很多。

参考资料
【1】https://www.ibm.com/developerworks/cn/java/j-lo-nio2/
【2】https://github.com/redkale/redkale
【3】http://colobu.com/2014/11/13/java-aio-introduction/
【4】http://zjumty.iteye.com/blog/1896350
【5】http://stevex.blog.51cto.com/4300375/1581701
【6】《Pro Java 7 NIO2》