# 3.6 編寫一個簡單的檔案日誌

專案地址：<https://github.com/EDDYCJY/go-gin-example>

## 涉及知識點

* 自定義 log。

## 本文目標

在上一節中，我們解決了API's可以任意訪問的問題，那麼我們現在還有一個問題，就是我們的日誌，都是輸出到控制檯上的，這顯然對於一個專案來說是不合理的，因此我們這一節簡單封裝`log`庫，使其支援簡單的檔案日誌！

## 新建`logging`包

我們在`pkg`下新建`logging`目錄，新建`file.go`和`log.go`檔案，寫入內容：

### 編寫`file`檔案

**1、 file.go：**

```go
package logging

import (
    "os"
    "time"
    "fmt"
    "log"
)

var (
    LogSavePath = "runtime/logs/"
    LogSaveName = "log"
    LogFileExt = "log"
    TimeFormat = "20060102"
)

func getLogFilePath() string {
    return fmt.Sprintf("%s", LogSavePath)
}

func getLogFileFullPath() string {
    prefixPath := getLogFilePath()
    suffixPath := fmt.Sprintf("%s%s.%s", LogSaveName, time.Now().Format(TimeFormat), LogFileExt)

    return fmt.Sprintf("%s%s", prefixPath, suffixPath)
}

func openLogFile(filePath string) *os.File {
    _, err := os.Stat(filePath)
    switch {
        case os.IsNotExist(err):
            mkDir()
        case os.IsPermission(err):
            log.Fatalf("Permission :%v", err)
    }

    handle, err := os.OpenFile(filePath, os.O_APPEND | os.O_CREATE | os.O_WRONLY, 0644)
    if err != nil {
        log.Fatalf("Fail to OpenFile :%v", err)
    }

    return handle
}

func mkDir() {
    dir, _ := os.Getwd()
    err := os.MkdirAll(dir + "/" + getLogFilePath(), os.ModePerm)
    if err != nil {
        panic(err)
    }
}
```
* `os.Stat`：返回檔案資訊結構描述檔案。如果出現錯誤，會返回`*PathError`

  ```
  type PathError struct {
    Op   string
    Path string
    Err  error
  }
  ```
* `os.IsNotExist`：能夠接受`ErrNotExist`、`syscall`的一些錯誤，它會返回一個布林值，能夠得知檔案不存在或目錄不存在
* `os.IsPermission`：能夠接受`ErrPermission`、`syscall`的一些錯誤，它會返回一個布林值，能夠得知許可權是否滿足
* `os.OpenFile`：呼叫檔案，支援傳入檔名稱、指定的模式呼叫檔案、檔案許可權，返回的檔案的方法可以用於I/O。如果出現錯誤，則為`*PathError`。

```
const (
    // Exactly one of O_RDONLY, O_WRONLY, or O_RDWR must be specified.
    O_RDONLY int = syscall.O_RDONLY // 以只读模式打开文件
    O_WRONLY int = syscall.O_WRONLY // 以只写模式打开文件
    O_RDWR   int = syscall.O_RDWR   // 以读写模式打开文件
    // The remaining values may be or'ed in to control behavior.
    O_APPEND int = syscall.O_APPEND // 在写入时将数据追加到文件中
    O_CREATE int = syscall.O_CREAT  // 如果不存在，则创建一个新文件
    O_EXCL   int = syscall.O_EXCL   // 使用O_CREATE时，文件必须不存在
    O_SYNC   int = syscall.O_SYNC   // 同步IO
    O_TRUNC  int = syscall.O_TRUNC  // 如果可以，打开时
)
```

* `os.Getwd`：返回與當前目錄對應的根路徑名
* `os.MkdirAll`：建立對應的目錄以及所需的子目錄，若成功則返回`nil`，否則返回`error`
* `os.ModePerm`：`const`定義`ModePerm FileMode = 0777`

### 編寫`log`檔案

**2、log.go**

```go
package logging

import (
    "log"
    "os"
    "runtime"
    "path/filepath"
    "fmt"
)

type Level int

var (
    F *os.File

    DefaultPrefix = ""
    DefaultCallerDepth = 2

    logger *log.Logger
    logPrefix = ""
    levelFlags = []string{"DEBUG", "INFO", "WARN", "ERROR", "FATAL"}
)

const (
    DEBUG Level = iota
    INFO
    WARNING
    ERROR
    FATAL
)

func init() {
    filePath := getLogFileFullPath()
    F = openLogFile(filePath)

    logger = log.New(F, DefaultPrefix, log.LstdFlags)
}

func Debug(v ...interface{}) {
    setPrefix(DEBUG)
    logger.Println(v)
}

func Info(v ...interface{}) {
    setPrefix(INFO)
    logger.Println(v)
}

func Warn(v ...interface{}) {
    setPrefix(WARNING)
    logger.Println(v)
}

func Error(v ...interface{}) {
    setPrefix(ERROR)
    logger.Println(v)
}

func Fatal(v ...interface{}) {
    setPrefix(FATAL)
    logger.Fatalln(v)
}

func setPrefix(level Level) {
    _, file, line, ok := runtime.Caller(DefaultCallerDepth)
    if ok {
        logPrefix = fmt.Sprintf("[%s][%s:%d]", levelFlags[level], filepath.Base(file), line)
    } else {
        logPrefix = fmt.Sprintf("[%s]", levelFlags[level])
    }

    logger.SetPrefix(logPrefix)
}
```
* `log.New`：建立一個新的日誌記錄器。`out`定義要寫入日誌資料的`IO`控制代碼。`prefix`定義每個生成的日誌行的開頭。`flag`定義了日誌記錄屬性

  ```
  func New(out io.Writer, prefix string, flag int) *Logger {
    return &Logger{out: out, prefix: prefix, flag: flag}
  }
  ```
* `log.LstdFlags`：日誌記錄的格式屬性之一，其餘的選項如下

  ```
  const (
    Ldate         = 1 << iota     // the date in the local time zone: 2009/01/23
    Ltime                         // the time in the local time zone: 01:23:23
    Lmicroseconds                 // microsecond resolution: 01:23:23.123123.  assumes Ltime.
    Llongfile                     // full file name and line number: /a/b/c/d.go:23
    Lshortfile                    // final file name element and line number: d.go:23. overrides Llongfile
    LUTC                          // if Ldate or Ltime is set, use UTC rather than the local time zone
    LstdFlags     = Ldate | Ltime // initial values for the standard logger
  )
  ```

當前目錄結構：

```
gin-blog/
├── conf
│   └── app.ini
├── main.go
├── middleware
│   └── jwt
│       └── jwt.go
├── models
│   ├── article.go
│   ├── auth.go
│   ├── models.go
│   └── tag.go
├── pkg
│   ├── e
│   │   ├── code.go
│   │   └── msg.go
│   ├── logging
│   │   ├── file.go
│   │   └── log.go
│   ├── setting
│   │   └── setting.go
│   └── util
│       ├── jwt.go
│       └── pagination.go
├── routers
│   ├── api
│   │   ├── auth.go
│   │   └── v1
│   │       ├── article.go
│   │       └── tag.go
│   └── router.go
├── runtime
```

我們自定義的`logging`包，已經基本完成了，接下來讓它接入到我們的專案之中吧。我們開啟先前包含`log`包的程式碼，如下：

1. 開啟`routers`目錄下的`article.go`、`tag.go`、`auth.go`。
2. 將`log`包的引用刪除，修改引用我們自己的日誌包為`github.com/EDDYCJY/go-gin-example/pkg/logging`。
3. 將原本的`log.Println(...)`改為`logging.Info(...)`。

例如`auth.go`檔案的修改內容：

```go
package api

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/astaxie/beego/validation"

    "github.com/EDDYCJY/go-gin-example/pkg/e"
    "github.com/EDDYCJY/go-gin-example/pkg/util"
    "github.com/EDDYCJY/go-gin-example/models"
    "github.com/EDDYCJY/go-gin-example/pkg/logging"
)
...
func GetAuth(c *gin.Context) {
    ...
    code := e.INVALID_PARAMS
    if ok {
        ...
    } else {
        for _, err := range valid.Errors {
                logging.Info(err.Key, err.Message)
            }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : data,
    })
}
```
## 驗證功能

修改檔案後，重啟服務，我們來試試吧！

取得到API的Token後，我們故意傳錯誤URL引數給介面，如：`http://127.0.0.1:8000/api/v1/articles?tag_id=0&state=9999999&token=eyJhbG..`

然後我們到`$GOPATH/gin-blog/runtime/logs`檢視日誌：

```
$ tail -f log20180216.log 
[INFO][article.go:79]2018/02/16 18:33:12 [state 状态只允许0或1]
[INFO][article.go:79]2018/02/16 18:33:42 [state 状态只允许0或1]
[INFO][article.go:79]2018/02/16 18:33:42 [tag_id 标签ID必须大于0]
[INFO][article.go:79]2018/02/16 18:38:39 [state 状态只允许0或1]
[INFO][article.go:79]2018/02/16 18:38:39 [tag_id 标签ID必须大于0]
```

日誌結構一切正常，我們的記錄模式都為`Info`，因此字首是對的，並且我們是入參有問題，也把錯誤記錄下來了，這樣排錯就很方便了！

至此，本節就完成了，這只是一個簡單的擴充套件，實際上我們線上專案要使用的檔案日誌，是更復雜一些，開動你的大腦 舉一反三吧！

## 參考

### 本系列示例程式碼

* [go-gin-example](https://github.com/EDDYCJY/go-gin-example)

## 關於

### 修改記錄

* 第一版：2018年02月16日釋出文章
* 第二版：2019年10月01日修改文章

## ？

如果有任何疑問或錯誤，歡迎在 [issues](https://github.com/EDDYCJY/blog) 進行提問或給予修正意見，如果喜歡或對你有所幫助，歡迎 Star，對作者是一種鼓勵和推進。

### 我的微信公眾號

![image](../images/3051dc7d3068-8d0b0c3a11e74efd5fdfd7910257e70b.jpg)
