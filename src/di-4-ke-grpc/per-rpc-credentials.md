# 4.8 對 RPC 方法做自定義認證

專案地址：<https://github.com/EDDYCJY/go-grpc-example>

## 前言

在前面的章節中，我們介紹了兩種（證書算一種）可全域性認證的方法：

1. [TLS 證書認證](https://github.com/EDDYCJY/blog/blob/master/grpc/grpc-tls.md)
2. [基於 CA 的 TLS 證書認證](https://github.com/EDDYCJY/blog/blob/master/grpc/ca-tls.md)
3. [Unary and Stream interceptor](https://github.com/EDDYCJY/blog/blob/master/grpc/interceptor.md)

而在實際需求中，常常會對某些模組的 RPC 方法做特殊認證或校驗。今天將會講解、實作這塊的功能點

## 課前知識

```go
type PerRPCCredentials interface {
    GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error)
    RequireTransportSecurity() bool
}
```
在 gRPC 中預設定義了 PerRPCCredentials，它就是本章節的主角，是 gRPC 預設提供用於自定義認證的介面，它的作用是將所需的安全認證資訊新增到每個 RPC 方法的上下文中。其包含 2 個方法：

* GetRequestMetadata：取得當前請求認證所需的元資料（metadata）
* RequireTransportSecurity：是否需要基於 TLS 認證進行安全傳輸

## 目錄結構

新建 simple\_token\_server/server.go 和 simple\_token\_client/client.go，目錄結構如下：

```
go-grpc-example
├── client
│   ├── simple_client
│   ├── simple_http_client
│   ├── simple_token_client
│   └── stream_client
├── conf
├── pkg
├── proto
├── server
│   ├── simple_http_server
│   ├── simple_server
│   ├── simple_token_server
│   └── stream_server
└── vendor
```

## gRPC

### Client

```go
package main

import (
    "context"
    "log"

    "google.golang.org/grpc"

    "github.com/EDDYCJY/go-grpc-example/pkg/gtls"
    pb "github.com/EDDYCJY/go-grpc-example/proto"
)

const PORT = "9004"

type Auth struct {
    AppKey    string
    AppSecret string
}

func (a *Auth) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
    return map[string]string{"app_key": a.AppKey, "app_secret": a.AppSecret}, nil
}

func (a *Auth) RequireTransportSecurity() bool {
    return true
}

func main() {
    tlsClient := gtls.Client{
        ServerName: "go-grpc-example",
        CertFile:   "../../conf/server/server.pem",
    }
    c, err := tlsClient.GetTLSCredentials()
    if err != nil {
        log.Fatalf("tlsClient.GetTLSCredentials err: %v", err)
    }

    auth := Auth{
        AppKey:    "eddycjy",
        AppSecret: "20181005",
    }
    conn, err := grpc.Dial(":"+PORT, grpc.WithTransportCredentials(c), grpc.WithPerRPCCredentials(&auth))
    ...
}
```
在 Client 端，重點實作 `type PerRPCCredentials interface` 所需的方法，關注兩點即可：

* struct Auth：GetRequestMetadata、RequireTransportSecurity
* grpc.WithPerRPCCredentials

### Server

```go
package main

import (
    "context"
    "log"
    "net"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/metadata"
    "google.golang.org/grpc/status"

    "github.com/EDDYCJY/go-grpc-example/pkg/gtls"
    pb "github.com/EDDYCJY/go-grpc-example/proto"
)

type SearchService struct {
    auth *Auth
}

func (s *SearchService) Search(ctx context.Context, r *pb.SearchRequest) (*pb.SearchResponse, error) {
    if err := s.auth.Check(ctx); err != nil {
        return nil, err
    }
    return &pb.SearchResponse{Response: r.GetRequest() + " Token Server"}, nil
}

const PORT = "9004"

func main() {
    ...
}

type Auth struct {
    appKey    string
    appSecret string
}

func (a *Auth) Check(ctx context.Context) error {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return status.Errorf(codes.Unauthenticated, "自定义认证 Token 失败")
    }

    var (
        appKey    string
        appSecret string
    )
    if value, ok := md["app_key"]; ok {
        appKey = value[0]
    }
    if value, ok := md["app_secret"]; ok {
        appSecret = value[0]
    }

    if appKey != a.GetAppKey() || appSecret != a.GetAppSecret() {
        return status.Errorf(codes.Unauthenticated, "自定义认证 Token 无效")
    }

    return nil
}

func (a *Auth) GetAppKey() string {
    return "eddycjy"
}

func (a *Auth) GetAppSecret() string {
    return "20181005"
}
```
在 Server 端就更簡單了，實際就是呼叫 `metadata.FromIncomingContext` 從上下文中取得 metadata，再在不同的 RPC 方法中進行認證檢查

### 驗證

重新啟動 server.go 和 client.go，得到以下結果：

```
$ go run client.go
2018/10/05 20:59:58 resp: gRPC Token Server
```

修改 client.go 的值，製造兩者不一致，得到無效結果：

```
$ go run client.go
2018/10/05 21:00:05 client.Search err: rpc error: code = Unauthenticated desc = invalid token
exit status 1
```

### 一個個加太麻煩

我相信你肯定會問一個個加，也太麻煩了吧？有這個想法的你，應當把 `type PerRPCCredentials interface` 做成一個攔截器（interceptor）

## 總結

本章節比較簡單，主要是針對 RPC 方法的自定義認證進行了介紹，如果是想做全域性的，建議是舉一反三從攔截器下手哦。

## 參考

### 本系列示例程式碼

* [go-grpc-example](https://github.com/EDDYCJY/go-grpc-example)
