# android am start的使用方法

最近研究pc与Android应用程序通过usb通信，顺带研究了一下怎么通过adb启动Android应用程序，于是乎看到了am命名（activity manager）。

先附上谷歌开发文档中的描述文档路径，里面比我这里讲得详细。

http://developer.android.com/tools/help/adb.html#IntentSpec

虽然里面讲得很详细，不过对于我这种菜鸟来说，还是花了些时间才理解，希望这些理解能对与我遇到相同疑惑的开发者们有帮助，下面进入正文。

    adb shell

 


这个命令很简单，也就是启动Android的shell程序而已。毕竟需要进入Android的shell后，才能使用Android的系统命令嘛，就像在Linux下，要想使用命令，
也得启动Linux的shell一样。
之后，就到了使用am命令的时候了。

    am <command>

这里应该不用解释了。
当然，除了在远程shell里面输入这些命名外，我们也可以不启动远程shell，可以采用下面的方式：

    adb shell am <command>

好了，下面正式介绍<command>这部分类容的，因为我主要是想研究怎么启动Android应用，所以就只研究了start命令了，下面详细的讲讲start命令。

    start [options] <INTENT>
    
    option：
    
    -D: Enable debugging.
    -W: Wait for launch to complete.
    --start-profiler <FILE>: Start profiler and send results to <FILE>.
    -P <FILE>: Like --start-profiler, but profiling stops when the app goes idle.
    -R: Repeat the activity launch <COUNT> times. Prior to each repeat, the top activity will be finished.
    -S: Force stop the target app before starting the activity.
    --opengl-trace: Enable tracing of OpenGL functions.
    --user <USER_ID> | current: Specify which user to run as; if not specified, then run as the current user.

命令选项这部分没有研究具体的用途，自己目前也不知道，就不说了，当然希望各位大牛们给小弟解释解释。
我就谈谈对<INTENT>部分的理解吧。

    -a <ACTION>

启动时，要执行的动作，如：android.intent.action.VIEW，我在这里刚开始没有理解他怎么用，网上大部分例子都是：

`adb shell am -a android.intent.action.VIEW -d http://www.baidu.com` 启动浏览器打开一个网站

其实，还有更一般的用法，就是可以通过这种方式启动我们自己的app，假设我们在我们的app的AndroidManifest.xml文件中的activity标签中加入了

        <intent-filter>
            <action android:name="android.intent.action.MY_APP" />
    
            <category android:name="android.intent.category.DEFAULT" />
        </intent-filter>

那么我们就可以通过：adb shell am start -a android.intent.action.MY_APP来启动我们的应用了，MY_APP随便取名字，不和系统的内置名字冲突就行了。

    -d <DATA_URI>

启动时，要传入的URI，如：http://www.baidu.com。到这里，我想我们应该能猜到这个参数的意义了吧，它无非就是在启动我们的activity时，传入URI参数，供app程序分析，并执行对应操作，假设我们的app没有处理这个参数，传进去也没有什么意义。

    -t <MIME_TYPE>

摘自别人的文章“我在写android资源管理器（文件浏览器）的时候，希望能在资源管理器的中实现打开文件的操作，此时就需要用到文件的MIME类型。”，这玩意传进去具体怎么用，我也不清楚。用到的时候再研究吧。这个参数

    -c <CATEGORY>

启动时，

    -n <COMPONENT>

直接启动一个组件，例如：adb shell am start -n com.example.app/.ExampleActivity

    -f <FLAGS>

无知道这玩意的用途呀，还没有接触这方面的知识。

    --esn <EXTRA_KEY>
    
    -e|--es <EXTRA_KEY> <EXTRA_STRING_VALUE>
    
    --ez <EXTRA_KEY> <EXTRA_BOOLEAN_VALUE>
    
    --ei <EXTRA_KEY> <EXTRA_INT_VALUE>
    
    --el <EXTRA_KEY> <EXTRA_LONG_VALUE>
    
    --ef <EXTRA_KEY> <EXTRA_FLOAT_VALUE>
    
    --eu <EXTRA_KEY> <EXTRA_URI_VALUE>
    
    --ecn <EXTRA_KEY> <EXTRA_COMPONENT_NAME_VALUE>
    
    --eia <EXTRA_KEY> <EXTRA_INT_VALUE>[,<EXTRA_INT_VALUE...]
    
    --ela <EXTRA_KEY> <EXTRA_LONG_VALUE>[,<EXTRA_LONG_VALUE...]
    
    --efa <EXTRA_KEY> <EXTRA_FLOAT_VALUE>[,<EXTRA_FLOAT_VALUE...]

上面的这些命令属于统一类型，我就找一个解释就行了，这里就找--es <EXTRA_KEY> <EXTRA_STRING_VALUE>举例了。

我们看名字就知道这里传入的是一个键值对了，自然这些数据到时候会填入启动activity的intent中，比如我们输入下面的命令：

    am start -a android.intent.action.MY_APP --es data mystringdata

接着我们在activity中的onCreate函数中加入一下代码，检查一下数据是否传入成功：

   

     protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            
            String string = getIntent().getStringExtra("data");
            
            if (null != string) {
                Log.d("zjh", string);    
            } else {
                Log.d("zjh", "无附加数据");    
            }
        }

实际验证得到，输出了mystringdata这个数据。

因此，以上命令就是用于填充intent中对应的数据值而已。

后面的那些命令这里就不分析了，主要是我暂时用不到，当然大家能补充一下也是极好的。

最后总结一下：

start 后面的<INTENT>部分，无非就是填充完善intent中的内容，当然里面有很多不同的内容，代表着不同的意思，也有不同的用途，这里只能怪自己脑袋不好使，看到<INTENT>没有第一时间联想到它的意义，希望这篇文章对大家有帮助。