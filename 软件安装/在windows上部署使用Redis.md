# 在windows上部署使用Redis

标签（空格分隔）： 软件安装

---

### 下载Redis
https://github.com/MSOpenTech/redis

### 启动Redis

    redis-server redis.windows.conf
    
    redis-server redis.windows.conf --maxmemory 200m

第二种方法就是修改配置文件redis.windows.conf

    maxmemory 209715200

注意单位是字节
### 部署Redis
将Redis安装成windows服务，开机自启动，命令如下：

    redis-server --service-install redis.windows.conf

安装好后启动：

    redis-server --service-start

停止：

    redis-server --service-stop

安装多个实例：

    redis-server --service-install –service-name redisService1 –port 10001
    redis-server --service-start –service-name redisService1
    redis-server --service-install –service-name redisService2 –port 10002
    redis-server --service-start –service-name redisService2
    redis-server --service-install –service-name redisService3 –port 10003
    redis-server --service-start –service-name redisService3

卸载：

    redis-server --service-uninstall
### 最后
推荐一个Redis可视化管理工具：Redis Desktop Manager

https://redisdesktop.com/download