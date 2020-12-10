# Android studio快速生成jni头文件

![image-20200724102603593](.\assets\image-20200724102603593.png)

```
jni create
Program: javah
Arguments: -classpath D:\softwares\Android\Sdk\platforms\android-29\android.jar;. -jni -encoding $FileEncoding$ -d $ModuleFileDir$/src/main/jni $FileClass$
Working Directory: $ModuleFileDir$\src\main\java
```

