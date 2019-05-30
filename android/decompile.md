# 手动

## 工具

1. apktool:<https://ibotpeaches.github.io/Apktool/>，安装参考<https://ibotpeaches.github.io/Apktool/install/>
2. dex2jar: <https://github.com/pxb1988/dex2jar>
3. jd-gui: <http://java-decompiler.github.io/>

## 获取资源文件

```shell
apktool d test.apk -f
```

## 反编译

1. 将apk改为zip或rar解压

2. 进入目录，运行如下命令后生成`classes_dex2jar.jar`

   ```shell
   dex2jar.bat classes.dex
   ```

## 查看源码

` jd-gui`打开`classes_dex2jar.jar`文件查看源码

# 自动

使用`jadx`自动反编译apk，<https://github.com/skylot/jadx>

