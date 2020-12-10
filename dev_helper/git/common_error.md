### git clone 遇到如下问题

```shell
git clone https://aosp.tuna.tsinghua.edu.cn/kernel/goldfish.git
Cloning into 'goldfish'...
remote: Counting objects: 6062659, done.
fatal: the remote end hung up unexpectedly 1.16 GiB | 29.00 KiB/s     
fatal: early EOF
fatal: index-pack failed
```


可输入如下命令解决
```shell
git config --global http.postBuffer 524288000
```