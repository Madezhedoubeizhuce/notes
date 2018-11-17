### 下载

https://www.rust-lang.org/zh-CN/install.html

![1542445075816](E:\WorkSpace\notes\notes\软件安装\assets\1542445075816.png)

到rust官网下载rustup-init.exe进行安装。也可以参考该页面的其他安装方法，进行离线安装，但是离线安装不包含rustup工具，建议使用rustup-init.exe安装。

### 安装

`rustup`安装`rustc`, `cargo`, `rustup`以及其他的标准工具到Cargo的`bin`目录，在Unix上位于`$HOME/.cargo/bin`，而Windows是在`%USERPROFILE%\.cargo\bin`目录，`cargo install`安装的rust程序和Cargo插件都在此目录。

如果你想要指定rustup的安装目录，你可以在运行rustup-init之前设置`ARGO_HOME` 和`RUSTUP_HOME`两个环境变量来指定rustup的安装目录及cargo的安装目录。除此，还可以通过设置`RUSTUP_DIST_SERVER` 和`RUSTUP_UPDATE_ROOT` 来指定国内服务器，加快下载速度。如下：

```shell
set RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
set RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
```

接着就可以启动rustup-init程序安装rust，期间会有以下指示，可以只选择默认的。

安装成功后可以运行如下命令查看是否安装成功

```shell
rustc --version
```



### Windows 注意事项

在 Windows 上， Rust 需要 Visual C++ 生成工具 2013 或更新版本的支持。获取 Visual C++ 生成工具最方便的方法是安装[Microsoft Visual C++ Build Tools 2017 ](https://www.visualstudio.com/downloads/#build-tools-for-visual-studio-2017)。或者，你可以 [安装](https://www.visualstudio.com/downloads/) Visual Studio 2017, Visual Studio 2015, 或 Visual Studio 2013 并在安装过程中选择安装 「C++ 工具」。

关于在 Windows 上配置 Rust 的更多信息请查看 [Windows-specific `rustup` 文档](https://github.com/rust-lang-nursery/rustup.rs/blob/master/README.md#working-with-rust-on-windows)。

### 修改Rust Crates 源

在 **$HOME/.cargo/config** 中添加如下内容：

```none
[source.crates-io]
replace-with = 'ustc'
[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"
```

### vscode环境配置

在vscode上搜索**Rust(rls)**和**Rust**插件，他们都依赖rls服务完成代码提示之类的功能。

RLS提供了一个在后台运行的服务器，提供了Rust编程的相关信息，包括IDE，编辑器和其它工具。它支持诸如“goto定义”，符号搜索，重新格式化和代码完成等功能，并支持重命名和重构。

#### 安装nightly编译器

```shell
rustup install nightly
```

#### 安装RLS

```shell
rustup component add rls-preview rust-analysis rust-src
```

####  安装racer

为了实现代码自动补齐，需要安装racer：

```shell
cargo +nightly install racer
```

（可选）设置`RUST_SRC_PATH` ，建议设置该环境变量，因为设置了该环境变量能够加速，但不设置的话racer会自动检测。

```shell
set RUST_SRC_PATH=%USERPROFILE%.rustup\toolchains\stable-x86_64-pc-windows-gnu\lib\rustlib\src\rust\src
```

linux上可以这样设置：

```shell
export RUST_SRC_PATH=/usr/local/src/rust/src
```

或者

```shell
 export RUST_SRC_PATH="$(rustc --print sysroot)/lib/rustlib/src/rust/src"
```

输入如下命令测试是否安装成功（会显示出一些提示）：

```shell
racer complete std::io::B
```

使用vscode打开rust文件，如果下方有如下标识，说明配置成功：

![1542447669509](E:\WorkSpace\notes\notes\软件安装\assets\1542447669509.png)

这时可以确认一下是否有代码提示及格式化等功能了。