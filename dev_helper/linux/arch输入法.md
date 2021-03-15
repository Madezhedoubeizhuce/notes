

**安装fcitx和一些依赖**

```shell
sudo pacman -S fcitx
sudo pacman -S fcitx-configtool
sudo pacman -S fcitx-gtk2 fcitx-gtk3 
sudo pacman -S fcitx-qt4 fcitx-qt5
```

若要一次性安装 Fcitx 主程序和相关的模块，可使用此命令:

```shell
pacman -S fcitx-im
pacman -S fcitx-configtool
```

fcitx-configtool 是一个图形界面的设置工具，你可以用上面的命令安装。

**配置**

使用 FCITX 之前,你必须先进行一些环境设定:

在/etc/environment加入

```
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
```