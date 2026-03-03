# 1.1 聊一聊，Go 的相對路徑問題

## 前言

`Golang` 中存在各種執行方式，如何**正確的引用檔案路徑**成為一個值得商議的問題

以 [gin-blog](https://github.com/EDDYCJY/go-gin-example) 為例，當我們在專案根目錄下，執行 `go run main.go` 時能夠正常執行（`go build`也是正常的）

```
[$ gin-blog]# go run main.go
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:    export GIN_MODE=release
 - using code:    gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /api/v1/tags              --> gin-blog/routers/api/v1.GetTags (3 handlers)
...
```

那麼在不同的目錄層級下，不同的方式執行，又是怎麼樣的呢，帶著我們的疑問去學習

## 問題

1、 go run 我們上移目錄層級，到 `$GOPATH/src` 下，執行 `go run gin-blog/main.go`

```
[$ src]# go run gin-blog/main.go
2018/03/12 16:06:13 Fail to parse 'conf/app.ini': open conf/app.ini: no such file or directory
exit status 1
```

2、 go build，執行 `./gin-blog/main`

```
[$ src]# ./gin-blog/main
2018/03/12 16:49:35 Fail to parse 'conf/app.ini': open conf/app.ini: no such file or directory
```

這時候你要打一個大大的問號，就是我的程式讀取到什麼地方去了

我們透過分析得知，**`Golang`的相對路徑是相對於執行命令時的目錄**；自然也就讀取不到了

## 思考

既然已經知道問題的所在點，我們就可以尋思做點什麼 : )

我們想到相對路徑是相對執行命令的目錄，那麼我們取得可執行檔案的地址，拼接起來不就好了嗎？

## 實踐

我們編寫**取得當前可執行檔案路徑的方法**

```go
import (
    "path/filepath"
    "os"
    "os/exec"
    "string"
)

func GetAppPath() string {
    file, _ := exec.LookPath(os.Args[0])
    path, _ := filepath.Abs(file)
    index := strings.LastIndex(path, string(os.PathSeparator))

    return path[:index]
}
```
將其放到啟動程式碼處檢視路徑

```
log.Println(GetAppPath())
```

我們分別執行以下兩個命令，檢視輸出結果 1、 go run

```
$ go run main.go
2018/03/12 18:45:40 /tmp/go-build962610262/b001/exe
```

2、 go build

```
$ ./main
2018/03/12 18:49:44 $GOPATH/src/gin-blog
```

## 剖析

我們聚焦在 `go run` 的輸出結果上，發現它是一個臨時檔案的地址，這是為什麼呢？

在`go help run`中，我們可以看到

```go
Run compiles and runs the main package comprising the named Go source files.
A Go source file is defined to be a file ending in a literal ".go" suffix.
```
也就是 `go run` 執行時會將檔案放到 `/tmp/go-build...` 目錄下，編譯並執行

因此`go run main.go`出現`/tmp/go-build962610262/b001/exe`結果也不奇怪了，因為它已經跑到臨時目錄下去執行可執行檔案了

這就已經很清楚了，那麼我們想想，會出現哪些問題呢

* 依賴相對路徑的檔案，出現路徑出錯的問題
* `go run` 和 `go build` 不一樣，一個到臨時目錄下執行，一個可手動在編譯後的目錄下執行，路徑的處理方式會不同
* 不斷`go run`，不斷產生新的臨時檔案

這其實就是**根本原因**了，因為 `go run` 和 `go build` 的編譯檔案執行路徑並不同，執行的層級也有可能不一樣，自然而然就出現各種讀取不到的奇怪問題了

## 解決方案

**一、取得編譯後的可執行檔案路徑**

1、 將設定檔案的相對路徑與`GetAppPath()`的結果相拼接，可解決`go build main.go`的可執行檔案跨目錄執行的問題（如：`./src/gin-blog/main`）

```go
import (
    "path/filepath"
    "os"
    "os/exec"
    "string"
)

func GetAppPath() string {
    file, _ := exec.LookPath(os.Args[0])
    path, _ := filepath.Abs(file)
    index := strings.LastIndex(path, string(os.PathSeparator))

    return path[:index]
}
```
但是這種方式，對於`go run`依舊無效，這時候就需要2來補救

2、 透過傳遞引數指定路徑，可解決`go run`的問題

```go
package main

import (
    "flag"
    "fmt"
)

func main() {
    var appPath string
    flag.StringVar(&appPath, "app-path", "app-path")
    flag.Parse()
    fmt.Printf("App path: %s", appPath)
}
```
執行

```
go run main.go --app-path "Your project address"
```

**二、增加`os.Getwd()`進行多層判斷**

參見 [beego](https://github.com/astaxie/beego/blob/master/config.go#L133-L146) 讀取 `app.conf` 的程式碼

該寫法可相容 `go build` 和在專案根目錄執行 `go run` ，但是若跨目錄執行 `go run` 就不行

**三、設定全域性系統變數**

我們可以透過`os.Getenv`來取得系統全域性變數，然後與相對路徑進行拼接

1、 設定專案工作區

簡單來說，就是設定專案（應用）的工作路徑，然後與設定檔案、日誌檔案等相對路徑進行拼接，達到相對的絕對路徑來保證路徑一致

參見 [gogs](https://github.com/gogits/gogs/blob/master/pkg/setting/setting.go#L351) 讀取`GOGS_WORK_DIR`進行拼接的程式碼

2、 利用系統自帶變數

簡單來說就是透過系統自帶的全域性變數，例如`$HOME`等，將設定檔案存放在`$HOME/conf`或`/etc/conf`下

這樣子就能更加固定的存放設定檔案，**不需要額外去設定一個環境變數**

（這點今早與一位SFer討論了一波，感謝）

## 拓展

`go test` 在一些場景下也會遇到路徑問題，因為`go test`只能夠在當前目錄執行，所以在執行測試用例的時候，你的執行目錄已經是測試目錄了

需要注意的是，如果採用取得外部引數的辦法，用 `os.args` 時，`go test -args` 和 `go run`、`go build` 會有命令列引數位置的不一致問題

## 小結

這三種解決方案，在目前可見的開源專案或介紹中都能找到這些的身影

優缺點也是顯而易見的，我認為應在**不同專案選定合適的解決方案**即可

建議大家不要強依賴讀取設定檔案的模組，應當將其“堆積木”化，**需要什麼設定才去註冊什麼設定變數**，可以解決一部分的問題

大家又有什麼想法呢，一起討論一波？
