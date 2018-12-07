# Introduction to D-Bus

​	D-Bus是一个进程间通信系统。

​	D-Bus是有状态的和基于连接的，比类似UDP的底层协议更智能。它将消息作为离散数据进行传输，而不是像TCP一样传输连续的数据流。D-Bus支持一对一消息传递和发布/订阅通信。

​	D-Bus传输结构化的数据，并且以二进制形式处理数据，能够处理各种整型、浮点型数字，字符串及复合类型等。因为D-Bus传输的数据不是原始字节数据，因此可以验证消息、拒绝错误的消息。

## Buses

​	D-Bus有两个主要组件：

1. 点对点通信库，可以被任意两个进程使用，以便在它们之间进行通信。

2. 一个`dbusdaemon`，即守护进程。


### Addresses

​	应用程序使用D-Bus时， 通常是作为客户端或者服务端。

​	总线地址通常是Unix域套接字的文件名，例如`"/tmp/.hiddensocket"`。

​	D-Bus地址指定服务端监听的路径。例如：地址 `unix:path=/tmp/abcdef`指定了服务端监听`/tmp/abcdef`，同时，客户端与服务端连接时也使用这个路径。

​	你可以选择使用bus daemon或者不使用，如果使用bus daemon，你的应用将于bus daemon进行通信。如果不使用bus daemon，你需要自己设置作为服务端的应用，并且需要让客户端获取到正确的服务端地址，此时，客户端和服务端直接通信，不需要通过bus daemon，但是这不常用。

### Configuration and Startup

​	总线守护进程使用`dbus-launch`命令启动，使用选项`--config-file`选项来指示描述正在启动的总线的配置文件。标准总线将`/etc/dbus-1/system.conf`和`/etc/dbus-1/session.conf`作为各自的配置文件。

### bus names

​	总线上的连接名称，而不是总线的名称。总线名称由一系列由点分隔的标识符组成，例如： `"com.acme.Foo"`和标识符本身可能包含字母，数字，短划线和下划线。据说该连接拥有其总线名称。

**unique connection name**

​	建立连接后，总线会立即为其分配一个不可变的总线名称，它以冒号开头，例如： `":34-907"`。

​	unique connection name在总线守护程序的生命周期中永远不会被重用，也就是说，给定的名称将始终引用同一个应用。

**well-known names**

​	应用可以申请一个*well-known names*，*well-known names*.必须由两个或多个以点分隔的元素组成：`"com.acme.PortableHole"`。

​	可以把unique connection name认为是IP地址，而well-known names是域名。

​	除了路由消息之外，bus name还用于跟踪生命周期。当应用退出（或崩溃）时，操作系统内核将关闭其与消息总线的连接。然后，消息总线发出通知消息，通知其他应用。

## Object Model

### Objects

#### object

​	在总线上进行通信的一端，D-Bus称其为对象。

​	总线以对象为中心进行通信，总线承载的任何消息都是以下三种之一：

1. 客户端进程发送给对象的请求。
2. 对请求的回复，从对象返回到请求进程。
3. 从对象广播的消息会发送到注册了该消息的的客户端。总线支持两种形式的通信，1:1，即请求 - 回复形式的通信。1:n，即发布 - 订阅形式的通信。

**object paths**

​	类似Unix风格的文件路径。如`"/org/kde/kspread/sheets/3/cells/4/5"`

**native object**

​	本地编程语言的对象，比如`java.lang.Object`

### Proxies

​	代理对象是一个native object，能够使用它方便的访问远程对象：

​	不使用代理时：

```c++
Message message = new Message("/remote/object/path", "MethodName", arg1, arg2);
Connection connection = getBusConnection();
connection.send(message);
Message reply = connection.waitForReply(message);
if (reply.isError()) {
  
} else {
	Object returnValue = reply.getReturnValue();
}
```

​	使用代理：

```c++
Proxy proxy = new Proxy(getBusConnection(), "/remote/object/path");
Object returnValue = proxy.MethodName(arg1, arg2);
```

### Methods

***method***

​	当客户端向对象发送请求时，它会将此请求视为在对象上调用方法。

### Signals	

***signals***

​	客户端进程可以注册感兴趣的信号，当对象发出信号时，所有感兴趣的客户端都会收到通知。信号没有回复。信号能够传递参数。

### Interfaces

***members***

​	对象上支持的方法及可能发射的信号都称为members。

***interfaces***

​	接口指定了所有的对象成员。

## recap

| **A...**    | **is identified by a(n)...** | **which looks like...**                                      | **and is chosen by...**                               |
| ----------- | ---------------------------- | ------------------------------------------------------------ | ----------------------------------------------------- |
| Bus         | address                      | `unix:path=/var/run/dbus/system_bus_socket`                  | system configuration                                  |
| Connection  | bus name                     | `:34-907` (*unique*) or `com.mycompany.TextEditor` (*well-known*) | D-Bus (*unique*) or the owning program (*well-known*) |
| Object      | path                         | `/com/mycompany/TextFileManager`                             | the owning program                                    |
| *Interface* | *interface name*             | `org.freedesktop.Hal.Manager`                                | *the owning program*                                  |
| Member      | member name                  | `ListNames`                                                  | the owning program                                    |

## Big Conceptual Picture

​	结合以上的所有概念，当你调用一个远程对象上的某个方法时，需要gei以下这些组件命名：

`Address -> [Bus Name] -> Path -> Interface -> Method`

​	其中Bus Name用中括号括起来，表示Bus Name是可选的，只有在使用bus daemon进行通信时需要指定Bus Name，如果直接连接到另一个应用时不需要指定Bus Name。

## Behind the Scenes

###  Messages

​	D-Bus通过在进程之间发送消息进行工作。一共有四种消息类型：

1. Method call消息，调用object上的一个method。
2. Method return消息，返回method调用的结果。
3. Error消息，返回method调用的错误信息。
4. Signal消息，通知，可以看作事件消息。

一次方法调用非常容易映射成消息：先发送了一个Method call消息，然后在响应中收到一个Method return消息或者一个Error 消息。

每条消息都有一个消息头*header*，里面包含*fields*；还有一个*body*，里面包含*arguments*。可以认为header是消息的路由信息，body是载荷。头部的field可能包含发送者的bus name，接受者的bus name，method或者signal名等。其中的一个头字段用于描述body中的类型信息。

### Calling a Method

​	DBus的方法调用由两个消息组成：由进程A发往进程B的method call消息，和由进程B发送给进程A的method reply消息。这两个消息都通过bus daemon路由。每个调用消息中都带有一个序列号，并且应答消息也包含这个序列号以便让调用者匹配相应的回复。

​	方法调用处理过程如下：

- 使用的dbus库可能提供proxy，就通过调用本地对象的一个方法触发远程对象的方法。如果是这样的话，那么应用调用proxy上的一个方法，然后proxy构造一个method call消息发送到远端进程中。
- 对于底层的dbus api，应用需要自己构造一个method call消息，无需使用代理。
- 任一情况下，method call消息包含：远端进程的bus name，方法名，方法参数，远端进程内的object path，以及可选的接口名称。
- method call消息发送给bus daemon中处理。
- bus daemon查看目标的bus name，如果有进程的bus name与之对应，那么bus daemon将这个method call消息转发到该进程。否则，bus daemon生成错误信息并作为method reply返回。
- 接收进程将method call消息解包。对于简单的底层api，可能立即执行方法并且发送一个method reply消息给bus daemon。当使用高层api时，可能会检查object path,interface和方法名，然后将method call消息转换成一个native object的方法进行调用，然后将执行结果作为method reply消息返回。
- bus daemon接收method reply消息，并且将其发送调用进程。
- 调用进程查看method reply消息，并且使用应答消息中的返回值，其中可能包含错误信息。当使用高层api时，应答消息可能被转换成了一个proxy方法的返回值，或者一个异常。

bus daemon 不会打乱消息的顺序。也就是说，如果你发送多个method call消息给同一个接收者的话，他们将按顺序被接收。但是，接收方不一定按顺序回应，比如接受者使用多个线程处理，就不能保证响应的顺序了。

### Emitting a Signal

​	signal是单向广播，其中可能包含参数，并没没有返回值。接收者想bus daemon注册感兴趣(`"match ruls"`)的信号，当接受到一个信号时， bus daemon将其发送给所有对该信号感兴趣的接受者。

​	DBus信号处理流程如下：

- 信号发送给bus daemon。
- signal消息中指定信号的接口名，发送进程的bus name，以及参数。
- 任何进程都可以注册`"match ruls"`（通常包括发送者和信号名称）来指示它所感兴趣的信号。
- bus daemon检查信号并找出对此信号感兴趣的进程，然后将该信号发送给这些进程。
- 收到信号的进程决定处理方式。

### Overview picture

![](.\assets\diagram.png)