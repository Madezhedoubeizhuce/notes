# MSYS2 镜像使用帮助

标签（空格分隔）： linux

---

## pacman 的配置
编辑 /etc/pacman.d/mirrorlist.mingw32 ，在文件开头添加：

    Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/mingw/i686

编辑 /etc/pacman.d/mirrorlist.mingw64 ，在文件开头添加：

    Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/mingw/x86_64

编辑 /etc/pacman.d/mirrorlist.msys ，在文件开头添加：

    Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/msys/$arch

然后执行

    pacman -Sy

刷新软件包数据即可。


## windows msys2终端窗口配置
在用户目录下添加.minttyrc文件：

    BoldAsFont=-1
    BackgroundColour=39,40,34
    CursorType=block
    Font=Consolas
    FontHeight=12
    Locale=zh_CN
    Charset=UTF-8
    Columns=120
    Rows=30
    Term=xterm-256color
    FontWeight=400
    FontIsBold=no