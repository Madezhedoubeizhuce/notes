# linux 编译指定库、头文件的路径问题

标签（空格分隔）： linux

---

## 1. 为什么会出现undefined reference to 'xxxxx'错误？
首先这是链接错误，不是编译错误，也就是说如果只有这个错误，说明你的程序源码本身没有问题，是你用编译器编译时参数用得不对，你没有指定链接程序要用到得库，比如你的程序里用到了一些数学函数，那么你就要在编译参数里指定程序要链接数学库，方法是在编译命令行里加入-lm。

## 2.-l参数和-L参数
**-l参数**就是用来指定程序要链接的库，-l参数紧接着就是库名，那么库名跟真正的库文件名有什么关系呢？就拿数学库来说，他的库名是m，他的库文件名是libm.so，很容易看出，把库文件名的头lib和尾.so去掉就是库名了。
**-L参数**跟着的是库文件所在的目录名。再比如我们把libtest.so放在/aaa/bbb/ccc目录下，那链接参数就是-L/aaa/bbb/ccc -ltest另外，大部分libxxxx.so只是一个链接

## 3. -include和-I参数
**-include**用来包含头文件，但一般情况下包含头文件都在源码里用#include xxxxxx实现，-include参数很少用。
**-I参数**是用来指定头文件目录，/usr/include目录一般是不用指定的，gcc知道去那里找，但是如果头文件不在/usr/include里我们就要用-I参数指定了，比如头文件放在/myinclude目录里，那编译命令行就要加上-I/myinclude参数了，如果不加你会得到一个"xxxx.h: No such file or directory"的错误。-I参数可以用相对路径，比如头文件在当前目录，可以用-I.来指定

## 4.几个相关的环境变量
**PKG_CONFIG_PATH**：用来指定pkg-config用到的pc文件的路径，默认是            /usr/lib/pkgconfig，pc文件是文本文件，扩展名是.pc，里面定义开发包的安装路径，Libs参数和Cflags参数等等。
**CC**：用来指定c编译器。
**CXX**：用来指定cxx编译器。
**LIBS**：跟上面的--libs作用差不多。
**CFLAGS**:跟上面的--cflags作用差不多。
CC，CXX，LIBS，CFLAGS手动编译时一般用不上，在做configure时有时用到，一般情况下不用管。
环境变量设定方法：export ENV_NAME=xxxxxxxxxxxxxxxxx