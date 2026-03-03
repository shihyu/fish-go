# 3.7 優雅的重啟服務

專案地址：<https://github.com/EDDYCJY/go-gin-example>

## 知識點

* 訊號量的瞭解。
* 應用熱更新。

## 本文目標

在前面編寫案例程式碼時，我相信你會想到，每次更新完程式碼，更新完設定檔案後，就直接這麼 `ctrl+c` 真的沒問題嗎，`ctrl+c`到底做了些什麼事情呢？

在這一節中我們簡單講述 `ctrl+c` 背後的**訊號**以及如何在`Gin`中**優雅的重啟服務**，也就是對 `HTTP` 服務進行熱更新。

## ctrl + c

> 核心在某些情況下發送訊號，比如在程序往一個已經關閉的管道寫資料時會產生`SIGPIPE`訊號

在終端執行特定的組合鍵可以使系統傳送特定的訊號給此程序，完成一系列的動作

| 命令        | 訊號      | 含義                                                   |
| --------- | ------- | ---------------------------------------------------- |
| ctrl + c  | SIGINT  | 強制程序結束                                               |
| ctrl + z  | SIGTSTP | 任務中斷，程序掛起                                            |
| ctrl + \\ | SIGQUIT | 程序結束 和 `dump core`                                   |
| ctrl + d  |         | EOF                                                  |
|           | SIGHUP  | 終止收到該訊號的程序。若程式中沒有捕捉該訊號，當收到該訊號時，程序就會退出（常用於 重啟、重新載入程序） |

因此在我們執行`ctrl + c`關閉`gin`服務端時，**會強制程序結束，導致正在訪問的使用者等出現問題**

常見的 `kill -9 pid` 會發送 `SIGKILL` 訊號給程序，也是類似的結果

### 訊號

本段中反覆出現**訊號**是什麼呢？

訊號是 `Unix` 、類 `Unix` 以及其他 `POSIX` 相容的作業系統中程序間通訊的一種有限制的方式

它是一種非同步的通知機制，用來提醒程序一個事件（硬體異常、程式執行異常、外部發出訊號）已經發生。當一個訊號傳送給一個程序，作業系統中斷了程序正常的控制流程。此時，任何非原子操作都將被中斷。如果程序定義了訊號的處理函式，那麼它將被執行，否則就執行預設的處理函式

### 所有訊號

```
$ kill -l
 1) SIGHUP     2) SIGINT     3) SIGQUIT     4) SIGILL     5) SIGTRAP
 6) SIGABRT     7) SIGBUS     8) SIGFPE     9) SIGKILL    10) SIGUSR1
11) SIGSEGV    12) SIGUSR2    13) SIGPIPE    14) SIGALRM    15) SIGTERM
16) SIGSTKFLT    17) SIGCHLD    18) SIGCONT    19) SIGSTOP    20) SIGTSTP
21) SIGTTIN    22) SIGTTOU    23) SIGURG    24) SIGXCPU    25) SIGXFSZ
26) SIGVTALRM    27) SIGPROF    28) SIGWINCH    29) SIGIO    30) SIGPWR
31) SIGSYS    34) SIGRTMIN    35) SIGRTMIN+1    36) SIGRTMIN+2    37) SIGRTMIN+3
38) SIGRTMIN+4    39) SIGRTMIN+5    40) SIGRTMIN+6    41) SIGRTMIN+7    42) SIGRTMIN+8
43) SIGRTMIN+9    44) SIGRTMIN+10    45) SIGRTMIN+11    46) SIGRTMIN+12    47) SIGRTMIN+13
48) SIGRTMIN+14    49) SIGRTMIN+15    50) SIGRTMAX-14    51) SIGRTMAX-13    52) SIGRTMAX-12
53) SIGRTMAX-11    54) SIGRTMAX-10    55) SIGRTMAX-9    56) SIGRTMAX-8    57) SIGRTMAX-7
58) SIGRTMAX-6    59) SIGRTMAX-5    60) SIGRTMAX-4    61) SIGRTMAX-3    62) SIGRTMAX-2
63) SIGRTMAX-1    64) SIGRTMAX
```

## 怎樣算優雅

### 目的

* 不關閉現有連線（正在執行中的程式）
* 新的程序啟動並替代舊程序
* 新的程序接管新的連線
* 連線要隨時響應使用者的請求，當用戶仍在請求舊程序時要保持連線，新使用者應請求新程序，不可以出現拒絕請求的情況

### 流程

1、替換可執行檔案或修改設定檔案

2、傳送訊號量 `SIGHUP`

3、拒絕新連線請求舊程序，但要保證已有連線正常

4、啟動新的子程序

5、新的子程序開始 `Accet`

6、系統將新的請求轉交新的子程序

7、舊程序處理完所有舊連線後正常結束

## 實作優雅重啟

### endless

> Zero downtime restarts for golang HTTP and HTTPS servers. (for golang 1.3+)

我們藉助 [fvbock/endless](https://github.com/fvbock/endless) 來實作 `Golang HTTP/HTTPS` 服務重新啟動的零停機

`endless server` 監聽以下幾種訊號量：

* syscall.SIGHUP：觸發 `fork` 子程序和重新啟動
* syscall.SIGUSR1/syscall.SIGTSTP：被監聽，但不會觸發任何動作
* syscall.SIGUSR2：觸發 `hammerTime`
* syscall.SIGINT/syscall.SIGTERM：觸發伺服器關閉（會完成正在執行的請求）

`endless` 正正是依靠監聽這些**訊號量**，完成管控的一系列動作

#### 安裝

```
go get -u github.com/fvbock/endless
```

#### 編寫

開啟 [gin-blog](https://github.com/EDDYCJY/go-gin-example) 的 `main.go`檔案，修改檔案：

```go
package main

import (
    "fmt"
    "log"
    "syscall"

    "github.com/fvbock/endless"

    "gin-blog/routers"
    "gin-blog/pkg/setting"
)

func main() {
    endless.DefaultReadTimeOut = setting.ReadTimeout
    endless.DefaultWriteTimeOut = setting.WriteTimeout
    endless.DefaultMaxHeaderBytes = 1 << 20
    endPoint := fmt.Sprintf(":%d", setting.HTTPPort)

    server := endless.NewServer(endPoint, routers.InitRouter())
    server.BeforeBegin = func(add string) {
        log.Printf("Actual pid is %d", syscall.Getpid())
    }

    err := server.ListenAndServe()
    if err != nil {
        log.Printf("Server err: %v", err)
    }
}
```
`endless.NewServer` 返回一個初始化的 `endlessServer` 物件，在 `BeforeBegin` 時輸出當前程序的 `pid`，呼叫 `ListenAndServe` 將實際“啟動”服務

#### 驗證

**編譯**

```
$ go build main.go
```

**執行**

```
$ ./main
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
...
Actual pid is 48601
```

啟動成功後，輸出了`pid`為 48601；在另外一個終端執行 `kill -1 48601` ，檢驗先前服務的終端效果

```
[root@localhost go-gin-example]# ./main
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:    export GIN_MODE=release
 - using code:    gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /auth                     --> ...
[GIN-debug] GET    /api/v1/tags              --> ...
...

Actual pid is 48601

...

Actual pid is 48755
48601 Received SIGTERM.
48601 [::]:8000 Listener closed.
48601 Waiting for connections to finish...
48601 Serve() returning...
Server err: accept tcp [::]:8000: use of closed network connection
```

可以看到該命令已經掛起，並且 `fork` 了新的子程序 `pid` 為 `48755`

```
48601 Received SIGTERM.
48601 [::]:8000 Listener closed.
48601 Waiting for connections to finish...
48601 Serve() returning...
Server err: accept tcp [::]:8000: use of closed network connection
```

大致意思為主程序（`pid`為48601）接受到 `SIGTERM` 訊號量，關閉主程序的監聽並且等待正在執行的請求完成；這與我們先前的描述一致

**喚醒**

這時候在 `postman` 上再次訪問我們的介面，你可以驚喜的發現，他“復活”了！

```
Actual pid is 48755
48601 Received SIGTERM.
48601 [::]:8000 Listener closed.
48601 Waiting for connections to finish...
48601 Serve() returning...
Server err: accept tcp [::]:8000: use of closed network connection


$ [GIN] 2018/03/15 - 13:00:16 | 200 |     188.096µs |   192.168.111.1 | GET      /api/v1/tags...
```

這就完成了一次正向的流轉了

你想想，每次更新發布、或者修改設定檔案等，只需要給該程序傳送**SIGTERM訊號**，而不需要強制結束應用，是多麼便捷又安全的事！

#### 問題

`endless` 熱更新是採取建立子程序後，將原程序退出的方式，這點不符合守護程序的要求

### http.Server - Shutdown()

如果你的`Golang >= 1.8`，也可以考慮使用 `http.Server` 的 [Shutdown](https://golang.org/pkg/net/http/#Server.Shutdown) 方法

```go
package main

import (
    "fmt"
    "net/http"
    "context"
    "log"
    "os"
    "os/signal"
    "time"


    "gin-blog/routers"
    "gin-blog/pkg/setting"
)

func main() {
    router := routers.InitRouter()

    s := &http.Server{
        Addr:           fmt.Sprintf(":%d", setting.HTTPPort),
        Handler:        router,
        ReadTimeout:    setting.ReadTimeout,
        WriteTimeout:   setting.WriteTimeout,
        MaxHeaderBytes: 1 << 20,
    }

    go func() {
        if err := s.ListenAndServe(); err != nil {
            log.Printf("Listen: %s\n", err)
        }
    }()

    quit := make(chan os.Signal)
    signal.Notify(quit, os.Interrupt)
    <- quit

    log.Println("Shutdown Server ...")

    ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
    defer cancel()
    if err := s.Shutdown(ctx); err != nil {
        log.Fatal("Server Shutdown:", err)
    }

    log.Println("Server exiting")
}
```
## 小結

在日常的服務中，優雅的重啟（熱更新）是非常重要的一環。而 `Golang` 在 `HTTP` 服務方面的熱更新也有不少方案了，我們應該根據實際應用場景挑選最合適的

## 參考

### 本系列示例程式碼

* [go-gin-example](https://github.com/EDDYCJY/go-gin-example)

### 拓展閱讀

* [manners](https://github.com/braintree/manners)
* [graceful](https://github.com/tylerb/graceful)
* [grace](https://github.com/facebookgo/grace)
* [plugin: new package for loading plugins · golang/go@0cbb12f · GitHub](https://github.com/golang/go/commit/0cbb12f0bbaeb3893b3d011fdb1a270291747ab0)

## 關於

### 修改記錄

* 第一版：2018年02月16日釋出文章
* 第二版：2019年10月01日修改文章

## ？

如果有任何疑問或錯誤，歡迎在 [issues](https://github.com/EDDYCJY/blog) 進行提問或給予修正意見，如果喜歡或對你有所幫助，歡迎 Star，對作者是一種鼓勵和推進。

### 我的微信公眾號

![image](../images/3051dc7d3068-8d0b0c3a11e74efd5fdfd7910257e70b.jpg)
