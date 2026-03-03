# 4.6 Unary and Stream interceptor

專案地址：<https://github.com/EDDYCJY/go-grpc-example>

## 前言

我想在每個 RPC 方法的前或後做某些事情，怎麼做？

本章節將要介紹的攔截器（interceptor），就能幫你在合適的地方實作這些功能。

## 有幾種方法

在 gRPC 中，大類可分為兩種 RPC 方法，與攔截器的對應關係是：

* 普通方法：一元攔截器（grpc.UnaryInterceptor）
* 流方法：流攔截器（grpc.StreamInterceptor）

## 看一看

### grpc.UnaryInterceptor

```go
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption {
    return func(o *options) {
        if o.unaryInt != nil {
            panic("The unary server interceptor was already set and may not be reset.")
        }
        o.unaryInt = i
    }
}
```
函式原型：

```
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

透過檢視原始碼可得知，要完成一個攔截器需要實作 `UnaryServerInterceptor` 方法。形參如下：

* ctx context.Context：請求上下文
* req interface{}：RPC 方法的請求引數
* info \*UnaryServerInfo：RPC 方法的所有資訊
* handler UnaryHandler：RPC 方法本身

### grpc.StreamInterceptor

```go
func StreamInterceptor(i StreamServerInterceptor) ServerOption
```
函式原型：

```
type StreamServerInterceptor func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
```

StreamServerInterceptor 與 UnaryServerInterceptor 形參的意義是一樣，不再贅述

### 如何實作多個攔截器

另外，可以發現 gRPC 本身居然只能設定一個攔截器，難道所有的邏輯都只能寫在一起？

關於這一點，你可以放心。採用開源專案 [go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware) 就可以解決這個問題，本章也會使用它。

```
import "github.com/grpc-ecosystem/go-grpc-middleware"

myServer := grpc.NewServer(
    grpc.StreamInterceptor(grpc_middleware.ChainStreamServer(
        ...
    )),
    grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
       ...
    )),
)
```

## gRPC

從本節開始編寫 gRPC interceptor 的程式碼，我們會將實作以下攔截器：

* logging：RPC 方法的入參出參的日誌輸出
* recover：RPC 方法的異常保護和日誌輸出

### 實作 interceptor

#### logging

```go
func LoggingInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    log.Printf("gRPC method: %s, %v", info.FullMethod, req)
    resp, err := handler(ctx, req)
    log.Printf("gRPC method: %s, %v", info.FullMethod, resp)
    return resp, err
}
```
#### recover

```go
func RecoveryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
    defer func() {
        if e := recover(); e != nil {
            debug.PrintStack()
            err = status.Errorf(codes.Internal, "Panic err: %v", e)
        }
    }()

    return handler(ctx, req)
}
```
### Server

```go
import (
    "context"
    "crypto/tls"
    "crypto/x509"
    "errors"
    "io/ioutil"
    "log"
    "net"
    "runtime/debug"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
    "google.golang.org/grpc/status"
    "google.golang.org/grpc/codes"
    "github.com/grpc-ecosystem/go-grpc-middleware"

    pb "github.com/EDDYCJY/go-grpc-example/proto"
)

...

func main() {
    c, err := GetTLSCredentialsByCA()
    if err != nil {
        log.Fatalf("GetTLSCredentialsByCA err: %v", err)
    }

    opts := []grpc.ServerOption{
        grpc.Creds(c),
        grpc_middleware.WithUnaryServerChain(
            RecoveryInterceptor,
            LoggingInterceptor,
        ),
    }

    server := grpc.NewServer(opts...)
    pb.RegisterSearchServiceServer(server, &SearchService{})

    lis, err := net.Listen("tcp", ":"+PORT)
    if err != nil {
        log.Fatalf("net.Listen err: %v", err)
    }

    server.Serve(lis)
}
```
## 驗證

### logging

啟動 simple\_server/server.go，執行 simple\_client/client.go 發起請求，得到結果：

```
$ go run server.go
2018/10/02 13:46:35 gRPC method: /proto.SearchService/Search, request:"gRPC" 
2018/10/02 13:46:35 gRPC method: /proto.SearchService/Search, response:"gRPC Server"
```

### recover

在 RPC 方法中人為地製造執行時錯誤，再重複啟動 server/client.go，得到結果：

#### client

```
$ go run client.go
2018/10/02 13:19:03 client.Search err: rpc error: code = Internal desc = Panic err: assignment to entry in nil map
exit status 1
```

#### server

```
$ go run server.go
goroutine 23 [running]:
runtime/debug.Stack(0xc420223588, 0x1033da9, 0xc420001980)
    /usr/local/Cellar/go/1.10.1/libexec/src/runtime/debug/stack.go:24 +0xa7
runtime/debug.PrintStack()
    /usr/local/Cellar/go/1.10.1/libexec/src/runtime/debug/stack.go:16 +0x22
main.RecoveryInterceptor.func1(0xc420223a10)
...
```

檢查服務是否仍然執行，即可知道 Recovery 是否成功生效

## 總結

透過本章節，你可以學會最常見的攔截器使用方法。接下來其它“新”需求只要舉一反三即可。

## 參考

### 本系列示例程式碼

* [go-grpc-example](https://github.com/EDDYCJY/go-grpc-example)
