## 添加未被检测到的有效分辨率

首先，运行`gtf`或者`cvt`，查询某分辨率的有效扫描频率。

```
 $ cvt 1280 1024
 
 # 1280x1024 59.89 Hz (CVT 1.31M4) hsync: 63.67 kHz; pclk: 109.00 MHz
 Modeline "1280x1024_60.00"  109.00  1280 1368 1496 1712  1024 1027 1034 1063 -hsync +vsync
```

然后通过--newmode参数新建一种xrandr模式，输入上面所得到的查询结果，其中Modeline关键词自然需要被省略。

```
   xrandr --newmode "1280x1024_60.00"  109.00  1280 1368 1496 1712  1024 1027 1034 1063 -hsync +vsync
```

新建模式后，我们需要把这模式添加到当前的输出设备（假定为VGA1）上。由于一些参数已经事先设置，只需输入模式名称即可，即1280x1024_60.00。

```
   xrandr --addmode VGA1 1280x1024_60.00
```

最后，再把VGA1的分辨率指定为刚刚添加的新模式。

```
   xrandr --output VGA1 --mode 1280x1024_60.00
```

注意，以上设置同样地只能在当前会话暂时生效。

### 1920x1080

```shell
xrandr --newmode "1920x1080_60.00" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync
xrandr --addmode Virtual1 "1920x1080_60.00"
```

### 使用 xorg.conf

可以通过 `/etc/X11/xorg.conf` 或 `/etc/xorg.conf` 配置 Xorg，用下面命令可以生成 `xorg.conf` 模板:

```
# Xorg :0 -configure
```

执行后会在 `/root/` 生成 `xorg.conf.new` 文件，然后你可以将它复制到 `/etc/X11/xorg.conf`。

**提示：** 如果已经运行了 X 服务器，请使用不同的 display，例如 `Xorg :2 -configure`。

## 使xrandr所更改的分辨率设置永久生效

使xrandr定制永久生效的方案有：

- `xorg.conf`（推荐）
- `.xprofile`
- kdm/gdm

### 在xorg.conf设置分辨率（推荐）

示例：

```
/etc/X11/xorg.conf
Section "Monitor"
    Identifier      "External DVI"
    Modeline        "1280x1024_60.00"  108.88  1280 1360 1496 1712  1024 1025 1028 1060  -HSync +Vsync
    Option          "PreferredMode" "1280x1024_60.00"
EndSection
Section "Device"
    Identifier      "ATI Technologies, Inc. M22 [Radeon Mobility M300]"
    Driver          "ati"
    Option          "Monitor-DVI-0" "External DVI"
EndSection
Section "Screen"
    Identifier      "Primary Screen"
    Device          "ATI Technologies, Inc. M22 [Radeon Mobility M300]"
    DefaultDepth    24
    SubSection "Display"
        Depth           24
        Modes   "1280x1024" "1024x768" "640x480"
    EndSubSection
EndSection

Section "ServerLayout"
        Identifier      "Default Layout"
        Screen          "Primary Screen"
EndSection
```

关于更多的配置细节，请阅读[Xorg (简体中文)](https://wiki.archlinux.org/index.php/Xorg_(简体中文))或[xorg.conf(5)](https://man.archlinux.org/man/xorg.conf.5)。