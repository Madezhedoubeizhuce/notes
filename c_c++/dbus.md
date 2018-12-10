# Introduction to D-Bus

​	DBus是一种IPC机制，由freedesktop.org项目提供，使用GPL许可证发行，用于进程间通信或进程与内核的通信。

​	DBus进程间通信主要有三层架构：

1. 底层接口层：主要是通过libdbus这个函数库，给予系统使用DBus的能力。 
2. 总线层：主要Message bus daemon这个总线守护进程提供的，在Linux系统启动时运行，负责进程间的消息路由和传递，其中包括Linux内核和Linux桌面环境的消息传递。总线守护进程可同时与多个应用程序相连，并能把来自一个应用程序的消息路由到0或者多个其他程序。 
3. 应用封装层：通过一系列基于特定应用程序框架将DBus的底层接口封装成友好的Wrapper库，供不同开发人员使用（DBus官方主页http://www.freedesktop.org/wiki/Software/dbus，提供了大部分编程语言的DBus库版本）。比如libdbus-glib, libdbus-python.	

## Overview picture

![](.\assets\diagram.png)

​	如上图所示，Bus Daemon Process是运行在linux系统中的一个后台守护进程，dbus-daemon运行时会调用libdus的库。Application Process1代表的就是应用程序进程，通过调用特定的应用程序框架的Wrapper库与dbus-daemon进行通信。
从上图也可以看出来Application和Daemon中其实还是通过socket进行通行。

​	DBus的三大优点：低延迟，低开销，高可用性。

- 低延迟：DBus一开始就是用来设计成避免来回传递和允许异步操作的。因此虽然在Application和Daemon之间是通过socket实现的，但是又去掉了socket的循环等待，保证了操作的实时高效。
- *低开销：DBus使用一个二进制的协议，不需要转化成像XML这样的文本格式。因为DBus是主要用来机器内部的IPC,而不是为了网络上的IPC机制而准备的.所以它才能够在本机内部达到最优效果。*
- 高可用性：DBus是基于消息机制而不是字节流机制。它能自动管理一大堆困难的IPC问题。同样的，DBus库被设计来让程序员能够使用他们已经写好的代码。而不会让他们放弃已经写好的代码，被迫通过学习新的IPC机制来根据新的IPC特性重写这些代码。

## Buses

### dbus-daemon

​	dbus-daemon是D-Bus消息总线守护进程(message bus daemon)。不同的进程通过连接到message bus daemon来进行通信。

​	有两种标准的消息总线：会话总线（Session Buses/per-user-login-session message bus）和系统总线（System Bus/systemwide message bus）。

#### 会话总线（Session Buses）

​	普通进程创建，可同时存在多条。会话总线属于某个进程私有，它用于进程间传递消息。

#### 系统总线（System Bus）

​	在引导时就会启动，它由操作系统和后台进程使用，安全性非常好，以使得任意的应用程序不能欺骗系统事件。其主要用于广播系统事件，例如更改打印机队列或添加/删除设备。当然，如果一个应用程序需要接受来自系统总线的消息，他也可以直接连接到系统总线中，但是他能发送的消息是受限的。


### Addresses

​	应用程序使用D-Bus时， 通常是作为客户端或者服务端。Addresses指明server将要监听的地方，client将要连接的地方

​	总线地址通常是Unix域套接字的文件名，例如`"/tmp/.hiddensocket"`。

​	D-Bus地址指定服务端监听的路径。例如：地址 `unix:path=/tmp/abcdef`指定了服务端监听`/tmp/abcdef`，同时，客户端与服务端连接时也使用这个路径。

​	你可以选择使用bus daemon或者不使用，如果使用bus daemon，你的应用将于bus daemon进行通信。如果不使用bus daemon，你需要自己设置作为服务端的应用，并且需要让客户端获取到正确的服务端地址，此时，客户端和服务端直接通信，不需要通过bus daemon，但是这不常用。

### bus names

​	总线上的连接名称，而不是总线的名称，主要是用来标识一个应用和消息总线的连接。总线名称由一系列由点分隔的标识符组成，例如： `"com.acme.Foo"`和标识符本身可能包含字母，数字，短划线和下划线。

**unique connection name**

​	建立连接后，总线会立即为其分配一个不可变的总线名称，它以冒号开头，例如： `":34-907"`。

​	unique connection name在总线守护程序的生命周期中永远不会被重用，也就是说，给定的名称将始终引用同一个应用。

**well-known names**

​	应用可以申请一个*well-known names*，*well-known names*.必须由两个或多个以点分隔的元素组成：`"com.acme.PortableHole"`。*well-known names*能够让其他进程快速方便的找到该进程所提供的服务。

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

​	对象上的方法及可能发射的信号都称为该对象的member。

***interfaces***

​	每个对象都有多个member，接口指定了对象的所有成员。

### Services

​	服务（Services）是进程注册的抽象。进程注册某个地址后，即可获得对应总线的服务。D-Bus提供了服务查询接口，进程可通过该接口查询某个服务是否存在。或者在服务结束时自动收到来自系统的消息。使用bus name的*well-known name*给service命名。

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
