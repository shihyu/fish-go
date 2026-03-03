# 5.2 Hello World

這節將開始編寫一個複雜的Hello World，涉及到許多的知識，建議大家認真思考其中的概念

## 需求

由於本實踐偏向`Grpc`+`Grpc Gateway`的方面，我們的需求是**同一個服務端支援`Rpc`和`Restful Api`**，那麼就意味著`http2`、`TLS`等等的應用，功能方面就是一個服務端能夠接受來自`grpc`和`Restful Api`的請求並響應

## 一、初始化目錄

我們先在$GOPATH中新建`grpc-hello-world`資料夾，我們專案的初始目錄目錄如下：

```
grpc-hello-world/
├── certs
├── client
├── cmd
├── pkg
├── proto
│   ├── google
│   │   └── api
└── server
```

* `certs`：證書憑證
* `client`：客戶端
* `cmd`：命令列
* `pkg`：第三方公共模組
* `proto`：`protobuf`的一些相關檔案（含`.proto`、`pb.go`、`.pb.gw.go`)，`google/api`中用於存放`annotations.proto`、`http.proto`
* `server`：服務端

## 二、製作證書

在服務端支援`Rpc`和`Restful Api`，需要用到`TLS`，因此我們要先製作證書

進入`certs`目錄，生成`TLS`所需的公鑰金鑰檔案

### 私鑰

```
openssl genrsa -out server.key 2048

openssl ecparam -genkey -name secp384r1 -out server.key
```

* `openssl genrsa`：生成`RSA`私鑰，命令的最後一個引數，將指定生成金鑰的位數，如果沒有指定，預設512
* `openssl ecparam`：生成`ECC`私鑰，命令為橢圓曲線金鑰引數生成及操作，本文中`ECC`曲線選擇的是`secp384r1`

### 自簽名公鑰

```
openssl req -new -x509 -sha256 -key server.key -out server.pem -days 3650
```

* `openssl req`：生成自簽名證書，`-new`指生成證書請求、`-sha256`指使用`sha256`加密、`-key`指定私鑰檔案、`-x509`指輸出證書、`-days 3650`為有效期，此後則輸入證書擁有者資訊

### 填寫資訊

```
Country Name (2 letter code) [XX]:
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:grpc server name
Email Address []:
```

## 三、`proto`

### 編寫

1、 `google.api`

我們看到`proto`目錄中有`google/api`目錄，它用到了`google`官方提供的兩個`api`描述檔案，主要是針對`grpc-gateway`的`http`轉換提供支援，定義了`Protocol Buffer`所擴充套件的`HTTP Option`

`annotations.proto`檔案：

```go
// Copyright (c) 2015, Google Inc.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

syntax = "proto3";

package google.api;

import "google/api/http.proto";
import "google/protobuf/descriptor.proto";

option java_multiple_files = true;
option java_outer_classname = "AnnotationsProto";
option java_package = "com.google.api";

extend google.protobuf.MethodOptions {
  // See `HttpRule`.
  HttpRule http = 72295728;
}
```
`http.proto`檔案：

```go
// Copyright 2016 Google Inc.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

syntax = "proto3";

package google.api;

option cc_enable_arenas = true;
option java_multiple_files = true;
option java_outer_classname = "HttpProto";
option java_package = "com.google.api";


// Defines the HTTP configuration for a service. It contains a list of
// [HttpRule][google.api.HttpRule], each specifying the mapping of an RPC method
// to one or more HTTP REST API methods.
message Http {
  // A list of HTTP rules for configuring the HTTP REST API methods.
  repeated HttpRule rules = 1;
}

// Use CustomHttpPattern to specify any HTTP method that is not included in the
// `pattern` field, such as HEAD, or "*" to leave the HTTP method unspecified for
// a given URL path rule. The wild-card rule is useful for services that provide
// content to Web (HTML) clients.
message HttpRule {
  // Selects methods to which this rule applies.
  //
  // Refer to [selector][google.api.DocumentationRule.selector] for syntax details.
  string selector = 1;

  // Determines the URL pattern is matched by this rules. This pattern can be
  // used with any of the {get|put|post|delete|patch} methods. A custom method
  // can be defined using the 'custom' field.
  oneof pattern {
    // Used for listing and getting information about resources.
    string get = 2;

    // Used for updating a resource.
    string put = 3;

    // Used for creating a resource.
    string post = 4;

    // Used for deleting a resource.
    string delete = 5;

    // Used for updating a resource.
    string patch = 6;

    // Custom pattern is used for defining custom verbs.
    CustomHttpPattern custom = 8;
  }

  // The name of the request field whose value is mapped to the HTTP body, or
  // `*` for mapping all fields not captured by the path pattern to the HTTP
  // body. NOTE: the referred field must not be a repeated field.
  string body = 7;

  // Additional HTTP bindings for the selector. Nested bindings must
  // not contain an `additional_bindings` field themselves (that is,
  // the nesting may only be one level deep).
  repeated HttpRule additional_bindings = 11;
}

// A custom pattern is used for defining custom HTTP verb.
message CustomHttpPattern {
  // The name of this custom HTTP verb.
  string kind = 1;

  // The path matched by this custom verb.
  string path = 2;
}
```
1. `hello.proto`

這一小節將編寫`Demo`的`.proto`檔案，我們在`proto`目錄下新建`hello.proto`檔案，寫入檔案內容：

```go
syntax = "proto3";

package proto;

import "google/api/annotations.proto";

service HelloWorld {
    rpc SayHelloWorld(HelloWorldRequest) returns (HelloWorldResponse) {
        option (google.api.http) = {
            post: "/hello_world"
            body: "*"
        };
    }
}

message HelloWorldRequest {
    string referer = 1;
}

message HelloWorldResponse {
    string message = 1;
}
```
在`hello.proto`檔案中，引用了`google/api/annotations.proto`，達到支援`HTTP Option`的效果

* 定義了一個`service`RPC服務`HelloWorld`，在其內部定義了一個`HTTP Option`的`POST`方法，`HTTP`響應路徑為`/hello_world`
* 定義`message`型別`HelloWorldRequest`、`HelloWorldResponse`，用於響應請求和返回結果

### 編譯

在編寫完`.proto`檔案後，我們需要對其進行編譯，就能夠在`server`中使用

進入`proto`目錄，執行以下命令

```
# 编译google.api
protoc -I . --go_out=plugins=grpc,Mgoogle/protobuf/descriptor.proto=github.com/golang/protobuf/protoc-gen-go/descriptor:. google/api/*.proto

#编译hello_http.proto为hello_http.pb.proto
protoc -I . --go_out=plugins=grpc,Mgoogle/api/annotations.proto=grpc-hello-world/proto/google/api:. ./hello.proto

#编译hello_http.proto为hello_http.pb.gw.proto
protoc --grpc-gateway_out=logtostderr=true:. ./hello.proto
```

執行完畢後將生成`hello.pb.go`和`hello.gw.pb.go`，分別針對`grpc`和`grpc-gateway`的功能支援

## 四、命令列模組 `cmd`

### 介紹

這一小節我們編寫命令列模組，為什麼要獨立出來呢，是為了將`cmd`和`server`兩者解耦，避免混淆在一起。

我們採用 [Cobra](https://github.com/spf13/cobra) 來完成這項功能，`Cobra`既是建立強大的現代CLI應用程式的庫，也是生成應用程式和命令檔案的程式。提供了以下功能：

* 簡易的子命令列模式
* 完全相容posix的命令列模式(包括短和長版本)
* 巢狀的子命令
* 全域性、本地和級聯`flags`
* 使用`Cobra`很容易的生成應用程式和命令，使用`cobra create appname`和`cobra add cmdname`
* 智慧提示
* 自動生成commands和flags的幫助資訊
* 自動生成詳細的help資訊`-h`，`--help`等等
* 自動生成的bash自動完成功能
* 為應用程式自動生成手冊
* 命令別名
* 定義您自己的幫助、用法等的靈活性。
* 可選與[viper](https://github.com/spf13/viper)緊密整合的apps

### 編寫`server`

在編寫`cmd`時需要先用`server`進行測試關聯，因此這一步我們先寫`server.go`用於測試

在`server`模組下 新建`server.go`檔案，寫入測試內容：

```go
package server

import (
    "log"
)

var (
    ServerPort string
    CertName string
    CertPemPath string
    CertKeyPath string
)

func Serve() (err error){
    log.Println(ServerPort)

    log.Println(CertName)

    log.Println(CertPemPath)

    log.Println(CertKeyPath)

    return nil
}
```
### 編寫`cmd`

在`cmd`模組下 新建`root.go`檔案，寫入內容：

```go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Use:   "grpc",
    Short: "Run the gRPC hello-world server",
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Println(err)
        os.Exit(-1)
    }
}
```
新建`server.go`檔案，寫入內容：

```go
package cmd

import (
    "log"

    "github.com/spf13/cobra"

    "grpc-hello-world/server"
)

var serverCmd = &cobra.Command{
    Use:   "server",
    Short: "Run the gRPC hello-world server",
    Run: func(cmd *cobra.Command, args []string) {
        defer func() {
            if err := recover(); err != nil {
                log.Println("Recover error : %v", err)
            }
        }()

        server.Serve()
    },
}

func init() {
    serverCmd.Flags().StringVarP(&server.ServerPort, "port", "p", "50052", "server port")
    serverCmd.Flags().StringVarP(&server.CertPemPath, "cert-pem", "", "./certs/server.pem", "cert pem path")
    serverCmd.Flags().StringVarP(&server.CertKeyPath, "cert-key", "", "./certs/server.key", "cert key path")
    serverCmd.Flags().StringVarP(&server.CertName, "cert-name", "", "grpc server name", "server's hostname")
    rootCmd.AddCommand(serverCmd)
}
```
我們在`grpc-hello-world/`目錄下，新建檔案`main.go`，寫入內容：

```go
package main

import (
    "grpc-hello-world/cmd"
)

func main() {
    cmd.Execute()
}
```
### 講解

要使用`Cobra`，按照`Cobra`標準要建立`main.go`和一個`rootCmd`檔案，另外我們有子命令`server`

1、`rootCmd`： `rootCmd`表示在沒有任何子命令的情況下的基本命令

2、`&cobra.Command`：

* `Use`：`Command`的用法，`Use`是一個行用法訊息
* `Short`：`Short`是`help`命令輸出中顯示的簡短描述
* `Run`：執行:典型的實際工作功能。大多數命令只會實作這一點；另外還有`PreRun`、`PreRunE`、`PostRun`、`PostRunE`等等不同時期的執行命令，但比較少用，具體使用時再檢視亦可

3、`rootCmd.AddCommand`：`AddCommand`向這父命令（`rootCmd`）新增一個或多個命令

4、`serverCmd.Flags().StringVarP()`：

一般來說，我們需要在`init()`函式中定義`flags`和處理設定，以`serverCmd.Flags().StringVarP(&server.ServerPort, "port", "p", "50052", "server port")`為例，我們定義了一個`flag`，值儲存在`&server.ServerPort`中，長命令為`--port`，短命令為`-p`，，預設值為`50052`，命令的描述為`server port`。這一種呼叫方式成為`Local Flags`

我們延伸一下，如果覺得每一個子命令都要設一遍覺得很麻煩，我們可以採用`Persistent Flags`：

`rootCmd.PersistentFlags().BoolVarP(&Verbose, "verbose", "v", false, "verbose output")`

作用：

`flag`是可以持久的，這意味著該`flag`將被分配給它所分配的命令以及該命令下的每個命令。對於全域性標記，將標記作為根上的持久標誌。

另外還有`Local Flag on Parent Commands`、`Bind Flags with Config`、`Required flags`等等，使用到再 [傳送](https://github.com/spf13/cobra#local-flag-on-parent-commands) 瞭解即可

### 測試

回到`grpc-hello-world/`目錄下執行`go run main.go server`，檢視輸出是否為（此時應為預設值）：

```
2018/02/25 23:23:21 50052
2018/02/25 23:23:21 dev
2018/02/25 23:23:21 ./certs/server.pem
2018/02/25 23:23:21 ./certs/server.key
```

執行`go run main.go server --port=8000 --cert-pem=test-pem --cert-key=test-key --cert-name=test-name`，檢驗命令列引數是否正確：

```
2018/02/25 23:24:56 8000
2018/02/25 23:24:56 test-name
2018/02/25 23:24:56 test-pem
2018/02/25 23:24:56 test-key
```

若都無誤，那麼恭喜你`cmd`模組的編寫正確了，下一部分開始我們的重點章節！

## 五、服務端模組 `server`

### 編寫`hello.go`

在`server`目錄下新建檔案`hello.go`，寫入檔案內容：

```go
package server

import (
    "golang.org/x/net/context"

    pb "grpc-hello-world/proto"
)

type helloService struct{}

func NewHelloService() *helloService {
    return &helloService{}
}

func (h helloService) SayHelloWorld(ctx context.Context, r *pb.HelloWorldRequest) (*pb.HelloWorldResponse, error) {
    return &pb.HelloWorldResponse{
        Message : "test",
    }, nil
}
```
我們建立了`helloService`及其方法`SayHelloWorld`，對應`.proto`的`rpc SayHelloWorld`，這個方法需要有2個引數：`ctx context.Context`用於接受上下文引數、`r *pb.HelloWorldRequest`用於接受`protobuf`的`Request`引數（對應`.proto`的`message HelloWorldRequest`）

### \*編寫`server.go`

這一小章節，我們編寫最為重要的服務端程式部分，涉及到大量的`grpc`、`grpc-gateway`及一些網路知識的應用

1、在`pkg`下新建`util`目錄，新建`grpc.go`檔案，寫入內容：

```go
package util

import (
    "net/http"
    "strings"

    "google.golang.org/grpc"
)

func GrpcHandlerFunc(grpcServer *grpc.Server, otherHandler http.Handler) http.Handler {
    if otherHandler == nil {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            grpcServer.ServeHTTP(w, r)
        })
    }
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.ProtoMajor == 2 && strings.Contains(r.Header.Get("Content-Type"), "application/grpc") {
            grpcServer.ServeHTTP(w, r)
        } else {
            otherHandler.ServeHTTP(w, r)
        }
    })
}
```
`GrpcHandlerFunc`函式是用於判斷請求是來源於`Rpc`客戶端還是`Restful Api`的請求，根據不同的請求註冊不同的`ServeHTTP`服務；`r.ProtoMajor == 2`也代表著請求必須基於`HTTP/2`

2、在`pkg`下的`util`目錄下，新建`tls.go`檔案，寫入內容：

```go
package util

import (
    "crypto/tls"
    "io/ioutil"
    "log"

    "golang.org/x/net/http2"
)

func GetTLSConfig(certPemPath, certKeyPath string) *tls.Config {
    var certKeyPair *tls.Certificate
    cert, _ := ioutil.ReadFile(certPemPath)
    key, _ := ioutil.ReadFile(certKeyPath)

    pair, err := tls.X509KeyPair(cert, key)
    if err != nil {
        log.Println("TLS KeyPair err: %v\n", err)
    }

    certKeyPair = &pair

    return &tls.Config{
        Certificates: []tls.Certificate{*certKeyPair},
        NextProtos:   []string{http2.NextProtoTLS},
    }
}
```
`GetTLSConfig`函式是用於取得`TLS`設定，在內部，我們讀取了`server.key`和`server.pem`這類證書憑證檔案

* `tls.X509KeyPair`：從一對`PEM`編碼的資料中解析公鑰/私鑰對。成功則返回公鑰/私鑰對
* `http2.NextProtoTLS`：`NextProtoTLS`是談判期間的`NPN/ALPN`協議，用於**HTTP/2的TLS設定**
* `tls.Certificate`：返回一個或多個證書，實質我們解析`PEM`呼叫的`X509KeyPair`的函式宣告就是`func X509KeyPair(certPEMBlock, keyPEMBlock []byte) (Certificate, error)`，返回值就是`Certificate`

總的來說該函式是用於處理從證書憑證檔案（PEM），最終取得`tls.Config`作為`HTTP2`的使用引數

3、修改`server`目錄下的`server.go`檔案，該檔案是我們服務裡的核心檔案，寫入內容：

```go
package server

import (
    "crypto/tls"
    "net"
    "net/http"
    "log"

    "golang.org/x/net/context"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
    "github.com/grpc-ecosystem/grpc-gateway/runtime"

    pb "grpc-hello-world/proto"
    "grpc-hello-world/pkg/util"
)

var (
    ServerPort string
    CertName string
    CertPemPath string
    CertKeyPath string
    EndPoint string
)

func Serve() (err error){
    EndPoint = ":" + ServerPort
    conn, err := net.Listen("tcp", EndPoint)
    if err != nil {
        log.Printf("TCP Listen err:%v\n", err)
    }

    tlsConfig := util.GetTLSConfig(CertPemPath, CertKeyPath)
    srv := createInternalServer(conn, tlsConfig)

    log.Printf("gRPC and https listen on: %s\n", ServerPort)

    if err = srv.Serve(tls.NewListener(conn, tlsConfig)); err != nil {
        log.Printf("ListenAndServe: %v\n", err)
    }

    return err
}

func createInternalServer(conn net.Listener, tlsConfig *tls.Config) (*http.Server) {
    var opts []grpc.ServerOption

    // grpc server
    creds, err := credentials.NewServerTLSFromFile(CertPemPath, CertKeyPath)
    if err != nil {
        log.Printf("Failed to create server TLS credentials %v", err)
    }

    opts = append(opts, grpc.Creds(creds))
    grpcServer := grpc.NewServer(opts...)

    // register grpc pb
    pb.RegisterHelloWorldServer(grpcServer, NewHelloService())

    // gw server
    ctx := context.Background()
    dcreds, err := credentials.NewClientTLSFromFile(CertPemPath, CertName)
    if err != nil {
        log.Printf("Failed to create client TLS credentials %v", err)
    }
    dopts := []grpc.DialOption{grpc.WithTransportCredentials(dcreds)}
    gwmux := runtime.NewServeMux()

    // register grpc-gateway pb
    if err := pb.RegisterHelloWorldHandlerFromEndpoint(ctx, gwmux, EndPoint, dopts); err != nil {
        log.Printf("Failed to register gw server: %v\n", err)
    }

    // http服务
    mux := http.NewServeMux()
    mux.Handle("/", gwmux)

    return &http.Server{
        Addr:      EndPoint,
        Handler:   util.GrpcHandlerFunc(grpcServer, mux),
        TLSConfig: tlsConfig,
    }
}
```
#### `server`流程剖析

我們將這一大塊程式碼，分成以下幾個部分來理解

**一、啟動監聽**

`net.Listen("tcp", EndPoint)`用於監聽本地的網路地址通知，它的函式原型`func Listen(network, address string) (Listener, error)`

引數：`network`必須傳入`tcp`、`tcp4`、`tcp6`、`unix`、`unixpacket`，若`address`為空或為0則會自動選擇一個埠號 返回值：透過檢視原始碼我們可以得知其返回值為`Listener`，結構體原型：

```go
type Listener interface {
    Accept() (Conn, error)
    Close() error
    Addr() Addr
}
```
透過分析得知，**最後`net.Listen`會返回一個監聽器的結構體，返回給接下來的動作，讓其執行下一步的操作**，它可以執行三類操作

* `Accept`：接受等待並將下一個連線返回給`Listener`
* `Close`：關閉`Listener`
* `Addr`：返回`Listener`的網路地址

**二、取得TLS**

透過`util.GetTLSConfig`解析得到`tls.Config`，傳達給`http.Server`服務的`TLSConfig`設定項使用

**三、建立內部服務**

`createInternalServer`函式，是整個服務端的核心流轉部分

程式採用的是`HTT2`、`HTTPS`也就是需要支援`TLS`，因此在啟動`grpc.NewServer`前，我們要將認證的中介軟體註冊進去

而前面所取得的`tlsConfig`僅能給`HTTP`使用，因此**第一步**我們要建立`grpc`的`TLS`認證憑證

**1、建立`grpc`的`TLS`認證憑證**

新增引用`google.golang.org/grpc/credentials`的第三方包，它實作了`grpc`庫支援的各種憑證，該憑證封裝了客戶機需要的所有狀態，以便與伺服器進行身份驗證並進行各種斷言，例如關於客戶機的身份，角色或是否授權進行特定的呼叫

我們呼叫`NewServerTLSFromFile`來達到我們的目的，它能夠從輸入證書檔案和伺服器的金鑰檔案**構造TLS證書憑證**

```go
func NewServerTLSFromFile(certFile, keyFile string) (TransportCredentials, error) {
    //LoadX509KeyPair读取并解析来自一对文件的公钥/私钥对
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, err
    }
    //NewTLS使用tls.Config来构建基于TLS的TransportCredentials
    return NewTLS(&tls.Config{Certificates: []tls.Certificate{cert}}), nil
}
```
**2、設定`grpc ServerOption`**

以`grpc.Creds(creds)`為例，其原型為`func Creds(c credentials.TransportCredentials) ServerOption`，該函式返回`ServerOption`，它為伺服器連線設定憑據

**3、建立`grpc`服務端**

函式原型：

```go
func NewServer(opt ...ServerOption) *Server
```
我們在此處建立了一個沒有註冊服務的`grpc`服務端，還沒有開始接受請求

```go
grpcServer := grpc.NewServer(opts...)
```

**4、註冊`grpc`服務**

```
pb.RegisterHelloWorldServer(grpcServer, NewHelloService())
```

**5、建立`grpc-gateway`關聯元件**

```go
ctx := context.Background()
dcreds, err := credentials.NewClientTLSFromFile(CertPemPath, CertName)
if err != nil {
    log.Println("Failed to create client TLS credentials %v", err)
}
dopts := []grpc.DialOption{grpc.WithTransportCredentials(dcreds)}
```

* `context.Background`：返回一個非空的空上下文。它沒有被登出，沒有值，沒有過期時間。它通常由主函式、初始化和測試使用，並作為傳入請求的**頂級上下文**
* `credentials.NewClientTLSFromFile`：從客戶機的輸入證書檔案構造TLS憑證
* `grpc.WithTransportCredentials`：設定一個連線級別的安全憑據(例：`TLS`、`SSL`)，返回值為`type DialOption`
* `grpc.DialOption`：`DialOption`選項設定我們如何設定連線（其內部具體由多個的`DialOption`組成，決定其設定連線的內容）

**6、建立`HTTP NewServeMux`及註冊`grpc-gateway`邏輯**

```go
gwmux := runtime.NewServeMux()

// register grpc-gateway pb
if err := pb.RegisterHelloWorldHandlerFromEndpoint(ctx, gwmux, EndPoint, dopts); err != nil {
    log.Println("Failed to register gw server: %v\n", err)
}

// http服务
mux := http.NewServeMux()
mux.Handle("/", gwmux)
```

* `runtime.NewServeMux`：返回一個新的`ServeMux`，它的內部對映是空的；`ServeMux`是`grpc-gateway`的一個請求多路複用器。它將`http`請求與模式匹配，並呼叫相應的處理程式
* `RegisterHelloWorldHandlerFromEndpoint`：如函式名，註冊`HelloWorld`服務的`HTTP Handle`到`grpc`端點
* `http.NewServeMux`：`分配并返回一个新的ServeMux`
* `mux.Handle`：為給定模式註冊處理程式

（帶著疑問去看程式）為什麼`gwmux`可以放入`mux.Handle`中？

首先我們看看它們的原型是怎麼樣的

（1）`http.NewServeMux()`

```go
func NewServeMux() *ServeMux {
        return new(ServeMux) 
}
```
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```
（2）`runtime.NewServeMux`？

```go
func NewServeMux(opts ...ServeMuxOption) *ServeMux {
    serveMux := &ServeMux{
        handlers:               make(map[string][]handler),
        forwardResponseOptions: make([]func(context.Context, http.ResponseWriter, proto.Message) error, 0),
        marshalers:             makeMarshalerMIMERegistry(),
    }
    ...
    return serveMux
}
```
（3）`http.NewServeMux()`的`Handle`方法

```go
func (mux *ServeMux) Handle(pattern string, handler Handler)
```

透過分析可得知，兩者`NewServeMux`都是最終返回`serveMux`，`Handler`中匯出的方法僅有`ServeHTTP`，功能是用於響應HTTP請求

我們回到`Handle interface`中，可以得出結論就是任何結構體，只要實作了`ServeHTTP`方法，這個結構就可以稱為`Handle`，`ServeMux`會使用該`Handler`呼叫`ServeHTTP`方法處理請求，這也就是**自定義`Handler`**

而我們這裡正是將`grpc-gateway`中註冊好的`HTTP Handler`無縫的植入到`net/http`的`Handle`方法中

**補充：在`go`中任何結構體只要實作了與介面相同的方法，就等同於實作了介面**

**7、註冊具體服務**

```go
if err := pb.RegisterHelloWorldHandlerFromEndpoint(ctx, gwmux, EndPoint, dopts); err != nil {
    log.Println("Failed to register gw server: %v\n", err)
}
```

在這段程式碼中，我們利用了前幾小節的

* 上下文
* `gateway-grpc`的請求多路複用器
* 服務網路地址
* 設定好的安全憑據

註冊了`HelloWorld`這一個服務

**四、建立tls.NewListener**

```go
func NewListener(inner net.Listener, config *Config) net.Listener {
    l := new(listener)
    l.Listener = inner
    l.config = config
    return l
}
```
`NewListener`將會建立一個`Listener`，它接受兩個引數，第一個是來自內部`Listener`的監聽器，第二個引數是`tls.Config`（必須包含至少一個證書）

**五、服務開始接受請求**

在最後我們呼叫`srv.Serve(tls.NewListener(conn, tlsConfig))`，可以得知它是`http.Server`的方法，並且需要一個`Listener`作為引數，那麼`Serve`內部做了些什麼事呢？

```go
func (srv *Server) Serve(l net.Listener) error {
    defer l.Close()
    ...

    baseCtx := context.Background() // base is always background, per Issue 16220
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    for {
        rw, e := l.Accept()
        ...
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(ctx)
    }
}
```
粗略的看，它建立了一個`context.Background()`上下文物件，並呼叫`Listener`的`Accept`方法開始接受外部請求，在取得到連線資料後使用`newConn`建立連線物件，在最後使用`goroutine`的方式處理連線請求，達到其目的

**補充：對於`HTTP/2`支援，在呼叫`Serve`之前，應將`srv.TLSConfig`初始化為提供的`Listener`的TLS設定。如果`srv.TLSConfig`非零，並且在`Config.NextProtos`中不包含字串`h2`，則不啟用`HTTP/2`支援**

## 六、驗證功能

### 編寫測試客戶端

在`grpc-hello-world/`下新建目錄`client`，新建`client.go`檔案，新增內容：

```go
package main

import (
    "log"

    "golang.org/x/net/context"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"

    pb "grpc-hello-world/proto"
)

func main() {
    creds, err := credentials.NewClientTLSFromFile("../certs/server.pem", "dev")
    if err != nil {
        log.Println("Failed to create TLS credentials %v", err)
    }
    conn, err := grpc.Dial(":50052", grpc.WithTransportCredentials(creds))
    defer conn.Close()

    if err != nil {
        log.Println(err)
    }

    c := pb.NewHelloWorldClient(conn)
    context := context.Background()
    body := &pb.HelloWorldRequest{
        Referer : "Grpc",
    }

    r, err := c.SayHelloWorld(context, body)
    if err != nil {
        log.Println(err)
    }

    log.Println(r.Message)
}
```
由於客戶端只是展示測試用，就簡單的來了，原本它理應歸類到`cobra`的管控下，設定管理等等都應可控化

在看這篇文章的你，可以試試將測試客戶端歸類好

### 啟動服務端

回到`grpc-hello-world/`目錄下，啟動服務端`go run main.go server`，成功則僅返回

```
2018/02/26 17:19:36 gRPC and https listen on: 50052
```

### 執行測試客戶端

回到`client`目錄下，啟動客戶端`go run client.go`，成功則返回

```
2018/02/26 17:22:57 Grpc
```

### 執行測試Restful Api

```
curl -X POST -k https://localhost:50052/hello_world -d '{"referer": "restful_api"}'
```

成功則返回`{"message":"restful_api"}`

## 最終目錄結構

```
grpc-hello-world
├── certs
│   ├── server.key
│   └── server.pem
├── client
│   └── client.go
├── cmd
│   ├── root.go
│   └── server.go
├── main.go
├── pkg
│   └── util
│       ├── grpc.go
│       └── tls.go
├── proto
│   ├── google
│   │   └── api
│   │       ├── annotations.pb.go
│   │       ├── annotations.proto
│   │       ├── http.pb.go
│   │       └── http.proto
│   ├── hello.pb.go
│   ├── hello.pb.gw.go
│   └── hello.proto
└── server
    ├── hello.go
    └── server.go
```

至此本節就結束了，推薦一下`jergoo`的文章，大家有時間可以看看

另外本節涉及了許多元件間的知識，值得大家細細的回味，非常有意義！

## 參考

### 示例程式碼

* [grpc-hello-world](https://github.com/EDDYCJY/grpc-hello-world)
