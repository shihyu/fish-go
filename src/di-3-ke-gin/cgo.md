# 3.18 Golang交叉編譯

專案地址：<https://github.com/EDDYCJY/go-gin-example>

## 知識點

* 跨平臺編譯

## 本文目標

在 [連載九](https://segmentfault.com/a/1190000013960558) 講解**構建Scratch映象**時，我們編譯可執行檔案用了另外一個形式的命令，如下：

```
$ CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o go-gin-example .
```

我想你可能會有疑問，今天本文會針對這塊進行講解。

## 說明

我們將講解命令各個引數的作用，希望你在閱讀時，將每一項串聯起來，你會發現這就是**交叉編譯相關的小知識**

也就是 `Golang` 令人心動的特性之一**跨平臺編譯**

### 一、CGO\_ENABLED

**作用：**

用於標識（宣告） `cgo` 工具是否可用

**意義：**

存在交叉編譯的情況時，`cgo` 工具是不可用的。在標準go命令的上下文環境中，交叉編譯意味著程式構建環境的目標計算架構的標識與程式執行環境的目標計算架構的標識不同，或者程式構建環境的目標作業系統的標識與程式執行環境的目標作業系統的標識不同

**小結：**

結合案例來說，我們是在宿主機編譯的可執行檔案，而在 `Scratch` 映象執行的可執行檔案；顯然兩者的計算機架構、執行環境標識你無法確定它是否一致（畢竟構建的 `docker` 映象還可以給他人使用），那麼我們就要進行交叉編譯，而交叉編譯不支援 `cgo`，因此這裡要停用掉它

關閉 `cgo` 後，在構建過程中會忽略 `cgo` 並靜態連結所有的依賴庫，而開啟 `cgo` 後，方式將轉為動態連結

**補充：**

`golang` 是預設開啟 `cgo` 工具的，可執行 `go env` 命令檢視

```
$ go env
GOARCH="amd64"
GOBIN=""
GOCACHE="/root/.cache/go-build"
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
...
GCCGO="gccgo"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
...
```

### 二、GOOS

用於標識（宣告）程式構建環境的目標作業系統

如：

* linux
* windows

### 三、GOARCH

用於標識（宣告）程式構建環境的目標計算架構

若不設定，預設值與程式執行環境的目標計算架構一致（案例就是採用的預設值）

如：

* amd64
* 386

| 系統          | GOOS    | GOARCH |
| ----------- | ------- | ------ |
| Windows 32位 | windows | 386    |
| Windows 64位 | windows | amd64  |
| OS X 32位    | darwin  | 386    |
| OS X 64位    | darwin  | amd64  |
| Linux 32位   | linux   | 386    |
| Linux 64位   | linux   | amd64  |

### 四、GOHOSTOS

用於標識（宣告）程式執行環境的目標作業系統

### 五、GOHOSTARCH

用於標識（宣告）程式執行環境的目標計算架構

### 六、go build

#### -a

強制重新編譯，簡單來說，就是不利用快取或已編譯好的部分檔案，直接所有包都是最新的程式碼重新編譯和關聯

#### -installsuffix

**作用：**

在軟體包安裝的目錄中**增加字尾標識**，以保持輸出與預設版本分開

**補充：**

如果使用 `-race` 標識，則字尾就會預設設定為 `-race` 標識，用於區別 `race` 和普通的版本

#### -o

指定編譯後的可執行檔名稱

### 小結

大部分引數指令，都有一定關聯性，且與交叉編譯的知識點相關，可以好好品味一下

最後可以看看 `go build help` 加深瞭解

```go
$ go help build
usage: go build [-o output] [-i] [build flags] [packages]
...
    -a
        force rebuilding of packages that are already up-to-date.
    -n
        print the commands but do not run them.
    -p n
        the number of programs, such as build commands or
        test binaries, that can be run in parallel.
        The default is the number of CPUs available.
    -race
        enable data race detection.
        Supported only on linux/amd64, freebsd/amd64, darwin/amd64 and windows/amd64.
    -msan
        enable interoperation with memory sanitizer.
        Supported only on linux/amd64,
        and only with Clang/LLVM as the host C compiler.
    -v
        print the names of packages as they are compiled.
    -work
        print the name of the temporary work directory and
        do not delete it when exiting.
    -x
        print the commands.

    -asmflags '[pattern=]arg list'
        arguments to pass on each go tool asm invocation.
    -buildmode mode
        build mode to use. See 'go help buildmode' for more.
    -compiler name
        name of compiler to use, as in runtime.Compiler (gccgo or gc).
    -gccgoflags '[pattern=]arg list'
        arguments to pass on each gccgo compiler/linker invocation.
    -gcflags '[pattern=]arg list'
        arguments to pass on each go tool compile invocation.
    -installsuffix suffix
        a suffix to use in the name of the package installation directory,
        in order to keep output separate from default builds.
        If using the -race flag, the install suffix is automatically set to race
        or, if set explicitly, has _race appended to it. Likewise for the -msan
        flag. Using a -buildmode option that requires non-default compile flags
        has a similar effect.
    -ldflags '[pattern=]arg list'
        arguments to pass on each go tool link invocation.
    -linkshared
        link against shared libraries previously created with
        -buildmode=shared.
    -pkgdir dir
        install and load all packages from dir instead of the usual locations.
        For example, when building with a non-standard configuration,
        use -pkgdir to keep generated packages in a separate location.
    -tags 'tag list'
        a space-separated list of build tags to consider satisfied during the
        build. For more information about build tags, see the description of
        build constraints in the documentation for the go/build package.
    -toolexec 'cmd args'
        a program to use to invoke toolchain programs like vet and asm.
        For example, instead of running asm, the go command will run
        'cmd args /path/to/asm <arguments for asm>'.
...
```
## 參考

### 本系列示例程式碼

* [go-gin-example](https://github.com/EDDYCJY/go-gin-example)

### 書籍

* Go併發程式設計實戰 第二版

## 關於

### 修改記錄

* 第一版：2018年02月16日釋出文章
* 第二版：2019年10月01日修改文章

## ？

如果有任何疑問或錯誤，歡迎在 [issues](https://github.com/EDDYCJY/blog) 進行提問或給予修正意見，如果喜歡或對你有所幫助，歡迎 Star，對作者是一種鼓勵和推進。

### 我的微信公眾號

![image](../images/3051dc7d3068-8d0b0c3a11e74efd5fdfd7910257e70b.jpg)
