## VS Code中go插件安装

vs code安装了go插件后，打开go文件只有会提示缺少一些组件，但是点击安装所有往往会有自动安装一些go插件的时候总是会失败，此时vs code会在右下角提示Analysis Tools Missing，这时候只能手动安装这些组件了。

以下是手动执行的步骤：

1. 在`%GOPATH%\src\`目录下，建立路径`golang.org\x`

2. 进入到`%GOPATH%\src\golang.org\x`，下载需要工具的源码`git clone https://github.com/golang/tools.git tools`

3. clone完成后，会生成一个tools文件夹，这样工具所需要的源码已经准备好了

4. 进入到`%GOPATH%`下，执行

   ```powershell
   go install github.com/ramya-rao-a/go-outline
   go install github.com/acroca/go-symbols
   go install golang.org/x/tools/cmd/guru
   go install golang.org/x/tools/cmd/gorename
   go install github.com/rogpeppe/godef
   go install github.com/sqs/goreturns
   go install github.com/cweill/gotests/gotests
   ```

如果上面一样执行`go install github.com/golang/lint/golint`，会报错说无法找到golint，这是就需要如下处理了

1. 单独处理golint,golint的源码位于`https://github.com/golang/lint`,进入`%GOPATH%\src\golang.org\x`后执行`git clone https://github.com/golang/lint`下载golint需要的源码
2. 进入到`%GOPATH%`下，执行`go install github.com/golang/lint/golint`