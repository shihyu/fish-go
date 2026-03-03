# 1.3 用 Go 來了解一下 Redis 通訊協議

Go、PHP、Java... 都有那麼多包來支撐你使用 Redis，那你是否有想過

有了服務端，有了客戶端，他們倆是怎樣通訊，又是基於什麼通訊協議做出互動的呢？

## 介紹

基於我們的目的，本文主要講解和實踐 Redis 的通訊協議

Redis 的客戶端和服務端是透過 TCP 連線來進行資料互動， 伺服器預設的埠號為 6379

客戶端和伺服器傳送的命令或資料一律以 \r\n （CRLF）結尾（這是一條約定）

## 協議

在 Redis 中分為**請求**和**回覆**，而請求協議又分為新版和舊版，新版統一請求協議在 Redis 1.2 版本中引入，最終在 Redis 2.0 版本成為 Redis 伺服器通訊的標準方式

本文是基於新版協議來實作功能，不建議使用舊版（1.2 挺老舊了）。如下是新協議的各種範例：

### 請求協議

1、 格式示例

```
*<参数数量> CR LF
$<参数 1 的字节数量> CR LF
<参数 1 的数据> CR LF
...
$<参数 N 的字节数量> CR LF
<参数 N 的数据> CR LF
```

在該協議下所有傳送至 Redis 伺服器的引數都是二進位制安全（binary safe）的

2、列印示例

```
*3
$3
SET
$5
mykey
$7
myvalue
```

3、實際協議值

```
"*3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$7\r\nmyvalue\r\n"
```

這就是 Redis 的請求協議規範，按照範例1編寫客戶端邏輯，最終傳送的是範例3，相信你已經有大致的概念了，Redis 的協議非常的簡潔易懂，這也是好上手的原因之一，你可以想想協議這麼定義的好處在哪？

### 回覆

Redis 會根據你請求協議的不同（執行的命令結果也不同），返回多種不同型別的回覆。在這個回覆“協議”中，可以透過檢查第一個位元組，確定這個回覆是什麼型別，如下：

* 狀態回覆（status reply）的第一個位元組是 "+"
* 錯誤回覆（error reply）的第一個位元組是 "-"
* 整數回覆（integer reply）的第一個位元組是 ":"
* 批量回復（bulk reply）的第一個位元組是 "$"
* 多條批量回復（multi bulk reply）的第一個位元組是 "\*"

有了回覆的頭部標識，結尾的 CRLF，你可以大致猜想出回覆“協議”是怎麼樣的，但是實踐才能得出真理，斎知道怕是你很快就忘記了 😀

## 實踐

### 與 Redis 伺服器互動

```go
package main

import (
    "log"
    "net"
    "os"

    "github.com/EDDYCJY/redis-protocol-example/protocol"
)

const (
    Address = "127.0.0.1:6379"
    Network = "tcp"
)

func Conn(network, address string) (net.Conn, error) {
    conn, err := net.Dial(network, address)
    if err != nil {
        return nil, err
    }

    return conn, nil
}

func main() {
        // 读取入参
    args := os.Args[1:]
    if len(args) <= 0 {
        log.Fatalf("Os.Args <= 0")
    }

        // 获取请求协议
    reqCommand := protocol.GetRequest(args)

    // 连接 Redis 服务器
    redisConn, err := Conn(Network, Address)
    if err != nil {
        log.Fatalf("Conn err: %v", err)
    }
    defer redisConn.Close()

        // 写入请求内容
    _, err = redisConn.Write(reqCommand)
    if err != nil {
        log.Fatalf("Conn Write err: %v", err)
    }

        // 读取回复
    command := make([]byte, 1024)
    n, err := redisConn.Read(command)
    if err != nil {
        log.Fatalf("Conn Read err: %v", err)
    }

        // 处理回复
    reply, err := protocol.GetReply(command[:n])
    if err != nil {
        log.Fatalf("protocol.GetReply err: %v", err)
    }

        // 处理后的回复内容
    log.Printf("Reply: %v", reply)
    // 原始的回复内容
    log.Printf("Command: %v", string(command[:n]))
}
```
在這裡我們完成了整個 Redis 客戶端和服務端互動的流程，分別如下：

1、讀取命令列引數：取得執行的 Redis 命令

2、取得請求協議引數

3、連線 Redis 伺服器，取得連線控制代碼

4、將請求協議引數寫入連線：傳送請求的命令列引數

5、從連線中讀取返回的資料：讀取先前請求的回覆資料

6、根據回覆“協議”內容，處理回覆的資料集

7、輸出處理後的回覆內容及原始回覆內容

### 請求

```go
func GetRequest(args []string) []byte {
    req := []string{
        "*" + strconv.Itoa(len(args)),
    }

    for _, arg := range args {
        req = append(req, "$"+strconv.Itoa(len(arg)))
        req = append(req, arg)
    }

    str := strings.Join(req, "\r\n")
    return []byte(str + "\r\n")
}
```
透過對 Redis 的請求協議的分析，可得出它的規律，先加上標誌位，計算引數總數量，再迴圈合併各個引數的位元組數量、值就可以了

### 回覆

```go
func GetReply(reply []byte) (interface{}, error) {
    replyType := reply[0]
    switch replyType {
    case StatusReply:
        return doStatusReply(reply[1:])
    case ErrorReply:
        return doErrorReply(reply[1:])
    case IntegerReply:
        return doIntegerReply(reply[1:])
    case BulkReply:
        return doBulkReply(reply[1:])
    case MultiBulkReply:
        return doMultiBulkReply(reply[1:])
    default:
        return nil, nil
    }
}

func doStatusReply(reply []byte) (string, error) {
    if len(reply) == 3 && reply[1] == 'O' && reply[2] == 'K' {
        return OkReply, nil
    }

    if len(reply) == 5 && reply[1] == 'P' && reply[2] == 'O' && reply[3] == 'N' && reply[4] == 'G' {
        return PongReply, nil
    }

    return string(reply), nil
}

func doErrorReply(reply []byte) (string, error) {
    return string(reply), nil
}

func doIntegerReply(reply []byte) (int, error) {
    pos := getFlagPos('\r', reply)
    result, err := strconv.Atoi(string(reply[:pos]))
    if err != nil {
        return 0, err
    }

    return result, nil
}

...
```
在這裡我們對所有回覆型別進行了分發，不同的回覆標誌位對應不同的處理方式，在這裡需求注意幾項問題，如下：

1、當請求的值不存在，會將特殊值 -1 用作回覆

2、伺服器傳送的所有字串都由 CRLF 結尾

3、多條批量回復是可基於批量回復的，要注意理解

4、無內容的多條批量回復是存在的

最重要的是，對不同回覆的規則的把控，能夠讓你更好的理解 Redis 的請求、回覆的互動過程 👌

## 小結

寫這篇文章的起因，是因為常常在使用 Redis 時，只是用，你不知道它是基於什麼樣的通訊協議來通訊，這樣的感覺是十分難受的

透過本文的講解，我相信你已經大致瞭解 Redis 客戶端是怎麼樣和服務端互動，也清楚了其所用的通訊原理，希望能夠對你有所幫助！

最後，如果想詳細檢視程式碼，右拐專案地址：<https://github.com/EDDYCJY/redis-protocol-example>

如果對你有所幫助，歡迎點個 Star 👍

## 參考

* [通訊協議](http://doc.redisfans.com/topic/protocol.html)
