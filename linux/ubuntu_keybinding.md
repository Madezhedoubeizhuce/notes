# ubuntu18.04下禁用Ctrl+Alt+Left/Right的方式（解决IDEA后退、前进快捷键的冲突）

## 1.查看是否被系统组合键占用
```shell
gsettings get org.gnome.desktop.wm.keybindings switch-to-workspace-left
```


返回：['<Control><Alt>Left']

结果：说明被 Control + Alt + Left 组合键占用

```shell
gsettings get org.gnome.desktop.wm.keybindings switch-to-workspace-left
```


返回：['<Control><Alt>Right'] 

结果：说明被 Control + Alt + Right 组合键占用

## 2.解除系统组合键占用
下面分别是针对：Ctrl+Alt+Left和Ctrl+Alt+Right

```shell
gsettings set org.gnome.desktop.wm.keybindings switch-to-workspace-left "[]"
gsettings set org.gnome.desktop.wm.keybindings switch-to-workspace-right "[]"
```

