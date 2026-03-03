# 8.2 log

## 日誌

### 輸出

```
2018/09/28 20:03:08 EDDYCJY Blog...
```

### 構成

\[日期]<空格>\[時分秒]<空格>\[內容]<\n>

## 原始碼剖析

### Logger

```go
type Logger struct {
    mu     sync.Mutex 
    prefix string
    flag   int
    out    io.Writer
    buf    []byte
}
```
1. mu：互斥鎖，用於確保原子的寫入
2. prefix：每行需寫入的日誌字首內容
3. flag：設定日誌輔助資訊（時間、檔名、行號）的寫入。可選如下標識位：

```
const (
    Ldate         = 1 << iota       // value: 1
    Ltime                           // value: 2
    Lmicroseconds                   // value: 4
    Llongfile                       // value: 8
    Lshortfile                      // value: 16
    LUTC                            // value: 32
    LstdFlags     = Ldate | Ltime   // value: 3
)
```

* Ldate：當地時區的格式化日期：2009/01/23
* Ltime：當地時區的格式化時間：01:23:23
* Lmicroseconds：在 Ltime 的基礎上，增加微秒的時間數值顯示
* Llongfile：完整的檔名和行號：/a/b/c/d.go:23
* Lshortfile：當前檔名和行號：d.go：23，會覆蓋 Llongfile 標識
* LUTC：如果設定 Ldate 或 Ltime，且設定 LUTC，則優先使用 UTC 時區而不是本地時區
* LstdFlags：Logger 的預設初始值（Ldate 和 Ltime）
* out：io.Writer
* buf：用於儲存將要寫入的日誌內容

### New

```go
func New(out io.Writer, prefix string, flag int) *Logger {
    return &Logger{out: out, prefix: prefix, flag: flag}
}

var std = New(os.Stderr, "", LstdFlags)
```
New 方法用於初始化 Logger，接受三個初始引數，可以定製化而在 log 包內預設會初始一個 std，它指向標準輸入流。而預設的標準輸出、標準錯誤就是顯示器（輸出到螢幕上），標準輸入就是鍵盤。輔助的時間資訊預設為 `Ldate | Ltime`，也就是 `2009/01/23 01:23:23`

```
// os
var (
    Stdin  = NewFile(uintptr(syscall.Stdin), "/dev/stdin")
    Stdout = NewFile(uintptr(syscall.Stdout), "/dev/stdout")
    Stderr = NewFile(uintptr(syscall.Stderr), "/dev/stderr")
)
```

* Stdin：標準輸入
* Stdout：標準輸出
* Stderr：標準錯誤

### Getter

* Flags
* Prefix

### Setter

* SetFlags
* SetPrefix
* SetOutput

### Prin&#x74;*, Fatal*, Panic\*

```go
func Print(v ...interface{}) {
    std.Output(2, fmt.Sprint(v...))
}

func Printf(format string, v ...interface{}) {
    std.Output(2, fmt.Sprintf(format, v...))
}

func Println(v ...interface{}) {
    std.Output(2, fmt.Sprintln(v...))
}

func Fatal(v ...interface{}) {
    std.Output(2, fmt.Sprint(v...))
    os.Exit(1)
}

func Panic(v ...interface{}) {
    s := fmt.Sprint(v...)
    std.Output(2, s)
    panic(s)
}

...
```
這一部分介紹最常用的日誌寫入方法，從原始碼可得知 `Xrintln`、`Xrintf` 函式 **換行**、**可變引數**都是透過 `fmt` 標準庫的方法去實作的

`Fatal` 和 `Panic` 是透過 `os.Exit(1)`、`panic(s)` 整合實作的。而具體的組裝邏輯是透過 `Output` 方法實作的

#### Logger.Output

```go
func (l *Logger) Output(calldepth int, s string) error {
    now := time.Now() // get this early.
    var file string
    var line int
    l.mu.Lock()
    defer l.mu.Unlock()
    if l.flag&(Lshortfile|Llongfile) != 0 {
        // Release lock while getting caller info - it's expensive.
        l.mu.Unlock()
        var ok bool
        _, file, line, ok = runtime.Caller(calldepth)
        if !ok {
            file = "???"
            line = 0
        }
        l.mu.Lock()
    }
    l.buf = l.buf[:0]
    l.formatHeader(&l.buf, now, file, line)
    l.buf = append(l.buf, s...)
    if len(s) == 0 || s[len(s)-1] != '\n' {
        l.buf = append(l.buf, '\n')
    }
    _, err := l.out.Write(l.buf)
    return err
}
```
Output 方法，簡單來講就是將寫入的日誌事件資訊組裝並輸出，它會根據 flag 標識位的不同來使用 `runtime.Caller` 去取得當前 goroutine 所執行的函式檔案、行號等呼叫資訊（log 標準庫中預設深度為 2）。另外如果結尾不是換行符 `\n`，將自動補全一個換行

#### Logger.formatHeader

```go
func (l *Logger) formatHeader(buf *[]byte, t time.Time, file string, line int) {
    *buf = append(*buf, l.prefix...)
    if l.flag&(Ldate|Ltime|Lmicroseconds) != 0 {
        if l.flag&LUTC != 0 {
            t = t.UTC()
        }
        if l.flag&Ldate != 0 {
            year, month, day := t.Date()
            itoa(buf, year, 4)
            *buf = append(*buf, '/')
            itoa(buf, int(month), 2)
            *buf = append(*buf, '/')
            itoa(buf, day, 2)
            *buf = append(*buf, ' ')
        }
        if l.flag&(Ltime|Lmicroseconds) != 0 {
            hour, min, sec := t.Clock()
            itoa(buf, hour, 2)
            *buf = append(*buf, ':')
            itoa(buf, min, 2)
            *buf = append(*buf, ':')
            itoa(buf, sec, 2)
            if l.flag&Lmicroseconds != 0 {
                *buf = append(*buf, '.')
                itoa(buf, t.Nanosecond()/1e3, 6)
            }
            *buf = append(*buf, ' ')
        }
    }
    if l.flag&(Lshortfile|Llongfile) != 0 {
        if l.flag&Lshortfile != 0 {
            short := file
            for i := len(file) - 1; i > 0; i-- {
                if file[i] == '/' {
                    short = file[i+1:]
                    break
                }
            }
            file = short
        }
        *buf = append(*buf, file...)
        *buf = append(*buf, ':')
        itoa(buf, line, -1)
        *buf = append(*buf, ": "...)
    }
}
```
該方法主要是用於格式化日誌頭（字首），根據入參不同的標識位，新增分隔符和對應的值到日誌資訊中。執行流程如下：

（1）如果不是空值，則將 prefix 寫入 buf

（2）如果設定 `Ldate`、`Ltime`、`Lmicroseconds`，則對應將日期和時間寫入 buf

（3）如果設定 `Lshortfile`、`Llongfile`，則對應將檔案和行號資訊寫入 buf

#### Logger.itoa

```go
func itoa(buf *[]byte, i int, wid int) {
    // Assemble decimal in reverse order.
    var b [20]byte
    bp := len(b) - 1
    for i >= 10 || wid > 1 {
        wid--
        q := i / 10
        b[bp] = byte('0' + i - q*10)
        bp--
        i = q
    }
    // i < 10
    b[bp] = byte('0' + i)
    *buf = append(*buf, b[bp:]...)
}
```
該方法主要用於將整數轉換為定長的十進位制 ASCII，同時給出負數寬度避免左側補 0。另外會以相反的順序組合十進位制

### 如何定製化 Logger

在標準庫內，可透過其開放的 New 方法來實作各種各樣的自定義 Logger 元件，但是為什麼也可以直接 `log.Print*` 等方法呢？

```go
func New(out io.Writer, prefix string, flag int) *Logger
```
其實是在標準庫內，如果你剛剛細心的看了前面的小節，不難發現其預設實作了一個 Logger 元件

```go
var std = New(os.Stderr, "", LstdFlags)
```

這也是一個小小的精妙之處 ⭕️

## 總結

透過查閱 log 標準庫的原始碼，可得知最簡單的一個日誌包應該如何編寫。另外 log 包是在所有涉及到 Logger 的地方都對 `sync.Mutex` 進行操作（以此解決原子問題），其餘邏輯均為組裝日誌資訊和轉換數值格式，該包較為經典，可以多讀幾遍 😄

## 問題

為什麼在呼叫 `runtime.Caller` 前要先解鎖，後再加鎖呢?
