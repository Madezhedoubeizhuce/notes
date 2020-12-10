# linux添加头文件路径

标签（空格分隔）： linux

---

对所有用户有效在/etc/profile增加以下内容。如果只对当前用户有效在Home目录下的.bashrc或.bash_profile里增加下面的内容：(注意：等号前面不要加空格,否则可能出现 command not found)
## 在PATH中找到可执行文件程序的路径。
`export PATH =$PATH:$HOME/bin`

## gcc找到头文件的路径
`C_INCLUDE_PATH=/usr/include/libxml2:/MyLib`
`export C_INCLUDE_PATH`

## g++找到头文件的路径
`CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:/usr/include/libxml2:/MyLib`
`export CPLUS_INCLUDE_PATH`

## 找到动态链接库的路径
`LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/MyLib`
`export LD_LIBRARY_PATH`

## 找到静态库的路径
`LIBRARY_PATH=$LIBRARY_PATH:/MyLib`
`export LIBRARY_PATH`