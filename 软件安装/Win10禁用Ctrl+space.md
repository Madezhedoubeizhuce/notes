

经过不断的尝试，终于在Win10上禁用了默认的Ctrl+space切换输入法

方法还是在这个链接里：[CTRL-Space always toggles Chinese IME (Windows 7)](https://link.zhihu.com/?target=http%3A//superuser.com/questions/327479/ctrl-space-always-toggles-chinese-ime-windows-7)

里面有个回答说到：

> HKEY_CURRENT_USER/Control Panel/Input Method/Hot Keys，保存的是**当前用户的快捷键配置**；
> HKEY_USERS\.DEFAULT\Control Panel\Input Method\Hot Keys，保存的是**默认的快捷键配置**；

在Win10上，不知为何，当前用户的快捷键配置并没有效果，每次重启之后，系统都会读取默认的快捷键配置。所以我们只要改默认的快捷键配置就行了~~

# Procedure

1. Go to `Start` > Type in `regedit` and start it
2. Navigate to `HKEY_CURRENT_USER/Control Panel/Input Method/Hot Keys`
3. Select the key named:
   - `00000070` for the `Chinese (Traditional) IME - Ime/NonIme Toggle` hotkey
   - `00000010` for the `Chinese (Simplified) IME - Ime/NonIme Toggle` hotkey
4. In the right sub-window, there are three subkeys.
   - Key Modifiers designate Alt/Ctrl/Shift/etc and is set to Ctrl (`02c00000`).
   - Virtual Key designates the finishing key and is set to Space (`20000000`).
5. Change the first byte in `Key Modifiers` from `02` to `00`
6. Change the first byte in `Virtual Key` from `20` to `FF`
7. Log off and log back on. I don't think it's necessary to restart.
8. Do not change the `Hot keys for input languages` in Control Panel, unless you want to do this all over again.