

# tcp粘包原因及粘包解决方式

近期在项目中的通信模块中发现tcp通信模块中，如果连续调用write发送数据的话，就会出现粘包现象，因为本来对tcp协议也不够熟悉，所以当时就大致猜测是因为tcp底层实现使用一个buffer将`socket.write()`的数据缓存起来，在某种条件下再将buffer中的数据真正的发出去，这样的话，在短时间内连续write的数据就会一起发送出去，从而造成粘包现象。

那么发生粘包的真正原因是什么呢？再查阅了相关的资料之后发现主要可能由下面两个原因造成：

1. 由于nagle算法导致的粘包：nagle算法会优化小数据包的发送，简单来说，就是将多个小数据包合并成一个包进行发送，详细流程会在下面进行介绍。
2. 由于接收端不及时导致粘包：如接收端的read线程被阻塞，tcp会先将接收到的数据保存在缓冲区中，这样在接收端进行read的时候可能缓冲区中已经接收了多个数据包，从而导致粘包，这与tcp的滑动窗口机制有关。

下面就分别来介绍一下nagle和滑动窗口机制。

## nagle算法

nagle算法有助于减少网络中报的数量，提升传输效率。考虑这样的场景，假设我们要使用tcp传输1字节的数据，tcp包头和ipv4包头加起来就有四十个字节，就是说整个包的大小为41字节，但是只有1字节是需要传输的数据，传输效率就非常低了。

nagle算法是这样要求的：

```
当一个连接中有在传数据（即那些已发送但未经确认的数据），小的报文段（长度小于SMSS）就不能发送，直到所有数据都收到ACK。并且，在收到ACK后，TCP需要手机这些小的数据，将其整合到一个报文段中发送。
```

也就是下面的算法，其中MSS是 maximum segment size：

```
if there is new data to send then
    if the window size ≥ MSS and available data is ≥ MSS then
        send complete MSS segment now
    else
        if there is unconfirmed data still in the pipe then
            enqueue data in the buffer until an acknowledge is received
        else
            send data immediately
        end if
    end if
end if
```

也就是说nagle算法在发送小的报文段的时候必须等到所有的数据都收到ack之后才真正发出去，如果和延时确认A一起工作就会出现问题。

延时确认就是利用累计ACK字段，允许TCP延时一段时间恢复ACK。

那么考虑这样的场景：

假设连续调用三次write方法发送一段10字节的数据，按照nagle算法，第一次调用write，数据会立即被发送，第二次调用的时候，由于延时确认，还没收到第一次调用的ack，所以会把数据加入到缓存中，第三次调用也相同，会把数据加入到缓存中，等到收到第一次发送的ACK时，就会将第二次和第三次的数据合并成一个tcp包一起发送。

来看一下下面的例子：

```java
try {
    File file = new File("login.pb");
    ByteArrayOutputStream bos = new ByteArrayOutputStream();
    FileInputStream fis = new FileInputStream(file);
    byte[] buf = new byte[1024];

    int c;
    while ((c = fis.read(buf)) != -1) {
        bos.write(buf, 0, c);
    }

    Socket socket = new Socket("172.16.101.61", 9999);

    socket.getOutputStream().write(data);
    socket.getOutputStream().write(data);
    socket.getOutputStream().write(data);

} catch (IOException e) {
    e.printStackTrace();
}
```

可以看到，这里连续进行三次写入，预想的情况应该是会发送两个包，下面抓包看看实际情况：

![image-20201228230418158](.\imgs\image-20201228230418158.png)

![image-20201228230523975](.\imgs\image-20201228230523975.png)

可以看出来，情况确实和我们分析的一致。

那到这里也就大致了解了nagle是怎么导致粘包的了，那么如何解决nagle导致的粘包呢？那就是设置TCP_NODELAY，把TCP_NODELAY设为true就会关闭nagle算法，从而确保小的报文能够立即被发送。还有一种是关闭延时确认，这在不同的操作系统上关闭方式不同。下面就介绍一下关闭TCP_NODELAY

的方式：

```java
try {
    File file = new File("login.pb");
    ByteArrayOutputStream bos = new ByteArrayOutputStream();
    FileInputStream fis = new FileInputStream(file);
    byte[] buf = new byte[1024];

    int c;
    while ((c = fis.read(buf)) != -1) {
        bos.write(buf, 0, c);
    }

    Socket socket = new Socket("172.16.101.61", 9999);
    socket.setTcpNoDelay(true);// 1

    socket.getOutputStream().write(data);
    socket.getOutputStream().write(data);
    socket.getOutputStream().write(data);

} catch (IOException e) {
    e.printStackTrace();
}
```

注释1处设置了TCP_NODELAY为true，下面看一下发送的包：

![image-20201228231751317](.\imgs\image-20201228231751317.png)

可以看出，这样就没有粘包现象了。