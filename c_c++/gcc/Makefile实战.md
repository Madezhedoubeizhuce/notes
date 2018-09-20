# Makefile实战

标签（空格分隔）： linux gcc

---

## makefile例子
相信在unix下编程的没有不知道makefile的，刚开始学习unix平台
下的东西，了解了下makefile的制作，觉得有点东西可以记录下。
　　下面是一个极其简单的例子：
现在我要编译一个Hello world，需要如下三个文件：

    print.h
    　　　　　　#include<stdio.h>
    　　　　　　void printhello();


    print.c
    　　　　　　#include"print.h"
    　　　　　　void printhello(){
    　　　　　　　　printf("Hello, world\n");
    　　　　　　}


    main.c
    　　　　　　#include "print.h"
    　　　　　　int main(void){
    　　　　　　　　printhello();
    　　　　　　　　return 0;
    　　　　　　}

　　想要编译成功需要如下步骤：
  1. 为每一个 *.c文件生成 *o文件。 　　
  2. 连接每一个*o文件，生成可执行文件。
  下面的makefile 就是根据这样的原则来写的。
  ###一：makefile 雏形：

makefile的撰写是基于规则的，当然这个规则也是很简单的，就是：

    target : prerequisites 
    　　command　　//任意的shell 命令

实例如下：

    makefile:
    　　　　helloworld : main.o print.o #helloword 就是我们要生成的目标
    　　　　　　　　　　　　　　　　　# main.o print.o是生成此目标的先决条件
    　　　　　　gcc -o helloworld main.o print.o#shell命令，最前面的一定是一个tab键
    
    　　　　mian.o : mian.c print.h
    　　　　　　gcc -c main.c
    　　　　print.o : print.c print.h
    　　　　　　gcc -c print.c
    　　　　
    　　　　clean :　　　　　　　　　　
    　　　　　　　　rm helloworld main.o print.o

　　OK，一个简单的makefile制作完毕，现成我们输入 make，自动调用Gcc编译了，
输入 make clean就会删除 hellowworld mian.o print.o

### 二：小步改进：


　　在上面的例子中我们可以发现 main.o print.o 被定义了多处，
我们是不是可以向C语言中定义一个宏一样定义它呢？当然可以：

    objects =  main.o print.o #应该叫变量的声明更合适
    
    　　　　helloworld : $(objects) //声明了变量以后使用就要$()了
    　　　　　　gcc -o helloworld$(objects)
    　　　  mian.o : mian.c print.h
    　　　　　　gcc -c main.c
    　　　　print.o : print.c print.h
    　　　　　　gcc -c print.c
    　　　　
    　　　　clean :　　　　　　　　　　
    　　　　　　　　rm helloworld $(objects)

修改完毕，这样使用了变量的话在很多文件的工程中就能体现出方便性了。

### 三：再进一步：
　　再看一下，为没一个*.o文件都写一句gcc -c main.c是不是显得多余了，
能不能把它干掉？而且 main.c 和print.c都需要print.h，为每一个都写上是
不是多余了，能不能再改进？
能，当然能了：

    objects =  main.o print.o
    
    　　　　helloworld : $(objects) 
    　　　　　　gcc -o helloworld$(objects)
    　　　　
    　　　　$(objects) : print.h # 都依赖print.h
    　　　  mian.o : mian.c  #干掉了gcc -c main.c 让Gun make自动推导了。
    　　　　print.o : print.c 　　　　
    　　　　clean :　　　　　　　　　　
    　　　　　　　　rm helloworld $(objects)

好了，一个简单的makefile就这样完毕了，简单吧。

## 通用的makefile

现在有一个libuv的demo文件uv_queue_work_test.c如下：

    #include <stdio.h>
    #include <uv.h>
    #include <unistd.h>
    
    #define FIB_UNTIL 10
    
    void fib(uv_work_t *req) {
        int n = *(int *) req->data;
        sleep(1);
        fprintf(stderr, "receive num %d\n", n);
    }
    
    void after_fib(uv_work_t *req, int status) {
        fprintf(stderr, "Done receive num %d\n", *(int *) req->data);
    }
    
    int main() {
    	uv_loop_t *loop;
        loop = uv_default_loop();
    
        int data;
        uv_work_t req;
    	
        data = 5;
    	req.data = (void *) &data;
    	uv_queue_work(loop, &req, fib, after_fib);
    	
        return uv_run(loop, UV_RUN_DEFAULT);
    }
这个文件依赖libuv.so和libuv中相关的库，根据上一节介绍的方法给这个文件写一个初步的makefile如下：

    objs=uv_queue_work_test.o
    test: $(objs)
    	g++ -o test $(objs) -luv -I/sdc/wangchenhui/local/include -L/sdc/wangchenhui/local/lib
    $(objs): uv_queue_work_test.cpp
    	g++ -c uv_queue_work_test.cpp
    clean:
    	rm test $(objs)

这样的makefile不灵活，而且不具备可复制性，如果工程下面有几十上百个源文件，那岂不是要一个一个写出来。

下面的makefile只需要稍作修改就可在各个工程之间通用

    SRCS= $(wildcard *.cpp)
    OBJS= $(SRCS:.cpp=.o)
    CC= g++
    INCLUDES=
    LIBS= -luv -L/sdc/wangchenhui/local/lib
    CCFLAGS= -g -Wall -O0
    
    test: $(OBJS)
    	$(CC) $^ -o $@ $(INCLUDES) $(LIBS)
    
    %.o: %.cpp
    	$(CC) -c $< $(CCFLAGS)
    clean:
    	rm *.o test
    .PHONY:clean

**SRCS= $(wildcard *.cpp)**
这条语句定义了一个变量SRCS，它的值就是当前面目录下面所有的以.c结尾的源文件。

**OBJS= $(SRCS:.cpp=.o)**
这里变量OBJS的值就是将SRCS里面所有.c文件编译出的.o目标文件

**CC= g++**
变量CC代表我们要使用的编译器

**INCLUDES=-I/xxx**
**LIBS= -luv -L/sdc/wangchenhui/local/lib**
这里指定除了编译器默认的头文件和库文件的路径之外需要额外引用的头文件路径以及库的路径（×××表示路径）。

**CCFLAGS = -g -Wall -O0**
CCFLAGS变量存放的是编译选项

**test: $(OBJS)**
**$(CC) $^ -o $@ $(INCLUDES) $(LIBS)**
my_app依赖于所有的.o文件，$^代表$(OBJS)，$@代表my_app

**%.o: %.cpp**
**$(CC) -c $< $(CCFLAGS)**
将所有的.c源代码编译成.o目标文件，这样写是不是很省事？
**clean:**
**rm *.o**
在执行make clean之后删除所有编译过程中生成的.o文件。

**.PHONY:clean**
每次执行make clean时都要执行rm *.o命令

## 自动变量

**＄＊**　不包含扩展名的目标文件名称
**＄＜**　第一个依赖文件的名称
**＄＠**　目标文件的完整名称
**＄＾**　所有不重复的目标依赖文件，以空格分开
．．．