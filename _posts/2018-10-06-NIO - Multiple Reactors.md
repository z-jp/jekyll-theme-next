---
layout: post
title: NIO - Multiple Reactors 模型
category: Java
tags: 
  - NIO
  - Java
---
### 前言
根据 [Scalable IO in Java - Doug Lea](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf) 实现了一个简单的 [NIO 服务器](https://github.com/zunpiau/Scalable-IO)，断断续续写了快一个月。比较麻烦的地方是多线程下 Selector 线程同步问题，在此记录。

### 单线程模型
先贴上单线程下的代码
```java
public class Reactor implements Runnable {

    /**
     * 每个 Reactor 对象 持有一个 Selector
     */
    final Selector selector;

    @Override
    public void run() {
        while (!Thread.interrupted()) {
            doSelect();
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()) {
                SelectionKey selectionKey = iterator.next();
                dispatch(selectionKey);
                iterator.remove();
            }
        }
    }

    /**
     * 模板，以便重写
     */
    int doSelect() throws IOException {
        return selector.select();
    }

    /**
     * 在 BasicReactor 中 selectionKey.attachment() 是 Acceptor、Handler 或者 Sender
     * 在 MultiReactor 中是 Acceptor
     * 在 SubReactor 中是 Handler 或者 Sender
     */
    private void dispatch(SelectionKey selectionKey) {
        ((Runnable) selectionKey.attachment()).run();
    }
}

public class BasicReactor extends Reactor {

    final ServerSocketChannel serverSocketChannel;

    BasicReactor(int port) throws IOException {
        serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        serverSocketChannel.bind(new InetSocketAddress(port));
        // 使用方法返回 Acceptor 的引用而不是直接 new Acceptor()
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT, getAcceptor());
    }

    /**
     * 使用方法返回 Acceptor 的引用，以便继承
     */
    Acceptor getAcceptor() {
        return new Acceptor();
    }

    /**
     * 处理 accept 事件
     */
    class Acceptor implements Runnable {

        /**
         * 创建 Handler 对象，注册通道等逻辑在 Handler 中实现
         */
        @Override
        public void run() {
            getHandler(BasicReactor.this, serverSocketChannel.accept());
        }

        /**
         * 理由同 getAcceptor()
         */
        Handler getHandler(Reactor reactor, SocketChannel socketChannel) throws IOException {
            return new Handler(reactor, socketChannel);
        }
    }

    /**
     * 处理 read 事件
     */
    class Handler implements Runnable {

        final SelectionKey key;
        private final Sender sender;

        /**
         * 传递 Reactor 而不是 Selector
         */
        Handler(Reactor reactor, SocketChannel socketChannel) throws IOException {
            sender = new Sender();
            socketChannel.configureBlocking(false);
            key = registerChannel(reactor, socketChannel);
            registerHandler();
        }

        @Override
        public void run() {
            read();
            // 数据读完，注册写事件
            if (inputIsComplete()) {
                sender.registerSender();
            }
        }

        /**
         * 分离 socketChannel 注册 和 设置 interestOps，以便重写
         */
        SelectionKey registerChannel(Reactor reactor, SocketChannel socketChannel) throws ClosedChannelException {
            return socketChannel.register(reactor.selector, 0);
        }

        void registerHandler() {
            key.attach(Handler.this);
            key.interestOps(SelectionKey.OP_READ);
            key.selector().wakeup();
        }

        /**
         * 处理 write 事件
         */
        class Sender implements Runnable {

            @Override
            public void run() {
                write();
                //数据写完，切换回写模式
                if (outputIsComplete()) {
                    registerHandler();
                }
            }

            void registerSender() {
                key.attach(this);
                key.interestOps(SelectionKey.OP_WRITE);
                key.selector().wakeup();
            }
        }
    }
}
```
单线程 Reactor 最大的问题是所有 Channel 都注册在一个 Selector，所有事件都在一个 select 循环中处理。一旦某个事件处理过慢就会影响其他事件。而 Selector 不能线程共享，因此只能考虑使用多个 Selector，服务器端 Channel 只处理 accept 事件，将创建的客户端 Channel 注册到不同 Selector 中。
### 多线程模型
```java
public class MultiReactor extends BasicReactor {

    /**
     * 多个子 reactor，处理读写事件
     */
    private final SubReactor[] subReactors;
    /**
     * 用于执行 SubReactor
     */
    private final ExecutorService rectorExecutor;

    @Override
    public void run() {
        for (SubReactor subReactor : subReactors) {
            rectorExecutor.submit(subReactor);
        }
        super.run();
    }

    /**
     * 返回当前类的 Acceptor 内部类
     */
    @Override
    Acceptor getAcceptor() {
        return new Acceptor();
    }

    class SubReactor extends Reactor {}

    class Acceptor extends BasicReactor.Acceptor {

        /**
         * 给 Handler 分配 SubReactor 而不是主 Reactor
         */
        @Override
        public void run() {
            try {
                SocketChannel socketChannel = serverSocketChannel.accept();
                SubReactor rector = subReactors[socketChannel.hashCode() % subReactors.length];
                getHandler(rector, socketChannel);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```
### 问题定位
40 多行代码中关键代码只有第 39 行，Acceptor 将客户端 Channel 交给子 Reactor 处理。但测试时发现客户端大概率收不到任何响应，通过 debug 发现是阻塞在了第一段 109 行中 `return socketChannel.register(reactor.selector, 0)`。Java doc 对这个方法有一句注释写道
> This method will then synchronize on the selector's key set and therefore may block if invoked concurrently with another registration or selection  operation involving the same selector.

而 selection  operation 也就是 `Selector#select()` 注释道：
> his method performs a blocking selection operation. It returns only after at least one channel is selected, this selector's {@link #wakeup wakeup} method is invoked, or the current thread is interrupted, whichever comes first.

也就是说 select 会阻塞 register，先来看看 `SocketChannel#register` 和 `Selector#select()` 两个方法的源码    
`SocketChannel#register` 方法调用的是子类`java.nio.channels.spi.AbstractSelectableChannel#register`
```java
public final SelectionKey register(Selector sel, int ops, Object att) throws ClosedChannelException {
    synchronized (regLock) {
        SelectionKey k = findKey(sel);
        if (k != null) {
            k.interestOps(ops);
            k.attach(att);
        }
        if (k == null) {
            // New registration
            synchronized (keyLock) {
                if (!isOpen())
                    throw new ClosedChannelException();
                k = ((AbstractSelector)sel).register(this, ops, att);
                addKey(k);
            }
        }
        return k;
    }
}
```
关键代码在第 13 行，实际的注册由 AbstractSelector 执行，而 `((AbstractSelector)sel).register(this, ops, att)` 调用的是子类 `sun.nio.ch.SelectorImpl#register` 
```java
protected final SelectionKey register(AbstractSelectableChannel ch, int ops, Object attachment) {
    SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);
    k.attach(attachment);
    synchronized (publicKeys) {
        implRegister(k);
    }
    k.interestOps(ops);
    return k;
}
```
可以看到 `SocketChannel#register` 最终需要获取到 Selector 的 `publicKeys` 锁。    
再来看`Selector#select()` 方法，实际调用的是子类 `sun.nio.ch.SelectorImpl#select(long)`
```java
public int select(long timeout) throws IOException {
    return lockAndDoSelect((timeout == 0) ? -1 : timeout);
}
```
继续往下看 `sun.nio.ch.SelectorImpl#lockAndDoSelect`
```java
private int lockAndDoSelect(long timeout) throws IOException {
    synchronized (this) {
        if (!isOpen())
            throw new ClosedSelectorException();
        synchronized (publicKeys) {
            synchronized (publicSelectedKeys) {
                return doSelect(timeout);
            }
        }
    }
}
```
`lockAndDoSelect()` 也 `publicKeys` 上做了同步，至此，阻塞的原因已经找到。`doSelect(timeout)` 在没有事件发生或被打断的情况下不会返回，导致 `SocketChannel#register` 一直获取不到锁而阻塞。而在单线程模型中 register 操作发生在本轮 select 返回之后，下一轮 select 之前。所以解决多线程下阻塞的思路就是：1. 结束当前 select 操作，2. 在下一轮 select 之前申请到锁，完成 register。实现方式有几种：
1. 使用 `Selector#select(long)` 的重载版本，select 会在超时后返回
2. register 前调用 `Selector#wakeup`，打断当前 select 操作   

问题在于，这两种方法只达成了目标1，不能保证接下来的 register 能获取到锁，只是减小了阻塞概率。因此需要在 select 前设置一个“门栓”。

### 最终代码
```java
public class MultiReactor extends BasicReactor {

    class SubReactor extends Reactor {

        private final ReentrantLock selectLock = new ReentrantLock();

        @Override
        int doSelect() throws IOException {
            // register 操作进行时阻塞
            selectLock.lock();
            // 立即释放，ReentrantLock 实际用作 Latch，但 CountDownLatch 没有 reset 方法
            selectLock.unlock();
            return super.doSelect();
        }
    }

    class Handler extends BasicReactor.Handler {

        /**
         * 传递 Reactor，携带 selectLock
         */
        SelectionKey registerChannel(Reactor reactor, SocketChannel socketChannel) throws ClosedChannelException {
            SubReactor subReactor = (SubReactor) reactor;
            try {
                // wakeup 前获取锁，阻止下一轮 select 操作
                subReactor.selectLock.lock();
                // 打断本轮 select 操作
                subReactor.selector.wakeup();
                return super.registerChannel(subReactor, socketChannel);
            } finally {
                subReactor.selectLock.unlock();
            }
        }
    }
}
```
### 小结
Java NIO 内部有大量的同步代码，多线程编程时稍不注意便有可能产生死锁问题。而且相对单线程，多个 Reactor 不能带来更快的处理速度，而是处理更多的连接。这点和 BIO 与 NIO 的对比是相似的。
完整代码：[GitHub](https://github.com/zunpiau/Scalable-IO)

### 参考链接
- [Java thread blocks while registering channel with selector while select() is called. What to do?](https://stackoverflow.com/questions/1057224/java-thread-blocks-while-registering-channel-with-selector-while-select-is-cal/1112809)   
- [How to detect a Selector.wakeup call](https://stackoverflow.com/questions/333593/how-to-detect-a-selector-wakeup-call)