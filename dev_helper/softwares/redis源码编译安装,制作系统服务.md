# redis源码编译安装,制作系统服务

标签（空格分隔）： linux redis

---
## 编译
下载redis源码包：redis-4.0.2.tar.gz
解压源码

```shell
tar -xzvf redis-4.0.2.tar.gz
```

进入解压开的文件夹

```shell
cd redis-4.0.2
```

编译

```shell
make
```

安装

```shell
make install
```

## 安装
进入utils文件夹

```shell
cd utils
```

运行install_server.sh脚本安装

```shell
./install_server.sh
```

安装过程会让你配置redis端口，配置文件名及路径等，无特殊需要的话可以都使用默认的
## 配置
可以使用已有redis.conf直接替换原有的，默认目录在/etc/redis下
dump.rdb替换有一个需要注意的，要先把redis的serviceting，默认目录在/var/lib/redis/下

启动：

```shell
service redis_3443 start
```

重启:

```shell
service redis_3443 restart
```

停止：

```shell
service redis_3443 stop
```