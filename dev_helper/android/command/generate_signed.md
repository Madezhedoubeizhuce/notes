# adb

将logcat命令重定向到本地：

```shell
adb logcat -v time > 1.txt
```

获取屏幕分辨率：

```shell
adb shell dumpsys window displays
```

获取当前activity

```shell
dumpsys activity activities
```

获取内存

```shell
cat /proc/meminfo
dumpsys meminfo
```

获取应用内存信息

```shell
dumpsys meminfo $package_name or $pid
```

查看签名文件信息：

```shell
keytool -list -v -keystore debug.keystore
```

# 查看apk的签名信息

1. 将apk解压；

2. 找到META-INF 下的.RSA文件；

3. 进入cmd环境，进入.RSA文件文件所在路径，命令：

   ```
   keytool -printcert -file XXX.RSA
   ```

   即可查看签名信息