# 5.3 Swagger瞭解一下

在[上一節](https://segmentfault.com/a/1190000013408485)，我們完成了一個服務端同時支援`Rpc`和`RESTful Api`後，你以為自己大功告成了，結果突然發現要寫`Api`文件和前端同事對接= = 。。。

你尋思有沒有什麼元件能夠自動化生成`Api`文件來解決這個問題，就在這時你發現了`Swagger`，一起了解一下吧！

## 介紹

### Swagger

`Swagger`是全球最大的`OpenAPI`規範（OAS）API開發工具框架，支援從設計和文件到測試和部署的整個API生命週期的開發

`Swagger`是目前最受歡迎的`RESTful Api`文件生成工具之一，主要的原因如下

* 跨平臺、跨語言的支援
* 強大的社群
* 生態圈 Swagger Tools（[Swagger Editor](https://github.com/swagger-api/swagger-editor)、[Swagger Codegen](https://github.com/swagger-api/swagger-codegen)、[Swagger UI](https://github.com/swagger-api/swagger-ui) ...）
* 強大的控制檯

同時`grpc-gateway`也支援`Swagger`

\[image]

### `OpenAPI`規範

`OpenAPI`規範是`Linux`基金會的一個專案，試圖透過定義一種用來描述API格式或API定義的語言，來規範`RESTful`服務開發過程。`OpenAPI`規範幫助我們描述一個API的基本資訊，比如：

* 有關該API的一般性描述
* 可用路徑（/資源）
* 在每個路徑上的可用操作（取得/提交...）
* 每個操作的輸入/輸出格式

目前V2.0版本的[OpenAPI規範](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md)（也就是SwaggerV2.0規範）已經發布並開源在github上。該文件寫的非常好，結構清晰，方便隨時查閱。

注：`OpenAPI`規範的介紹引用自[原文](https://huangwenchao.gitbooks.io/swagger/content/)

## 使用

### 生成`Swagger`的說明檔案

**第一**，我們需要檢查$GOBIN下是否包含`protoc-gen-swagger`可執行檔案

若不存在則需要執行：

```
go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-swagger
```

等待執行完畢後，可在`$GOPATH/bin`下發現該執行檔案，將其移動到`$GOBIN`下即可

**第二**，回到`$GOPATH/src/grpc-hello-world/proto`下，執行命令

```
protoc -I/usr/local/include -I. -I$GOPATH/src/grpc-hello-world/proto/google/api --swagger_out=logtostderr=true:. ./hello.proto
```

成功後執行`ls`即可看到`hello.swagger.json`檔案

### 下載`Swagger UI`檔案

`Swagger`提供視覺化的`API`管理平臺，就是[Swagger UI](https://github.com/swagger-api/swagger-ui)

我們將其原始碼下載下來，並將其`dist`目錄下的所有檔案複製到我們專案中的`$GOPATH/src/grpc-hello-world/third_party/swagger-ui`去

### 將`Swagger UI`轉換為`Go`原始碼

在這裡我們使用的轉換工具是[go-bindata](https://github.com/jteeuwen/go-bindata)

它支援將任何檔案轉換為可管理的`Go`原始碼。用於將二進位制資料嵌入到`Go`程式中。並且在將檔案資料轉換為原始位元組片之前，可以選擇壓縮檔案資料

#### 安裝

```
go get -u github.com/jteeuwen/go-bindata/...
```

完成後，將`$GOPATH/bin`下的`go-bindata`移動到`$GOBIN`下

#### 轉換

在專案下新建`pkg/ui/data/swagger`目錄，回到`$GOPATH/src/grpc-hello-world/third_party/swagger-ui`下，執行命令

```
go-bindata --nocompress -pkg swagger -o pkg/ui/data/swagger/datafile.go third_party/swagger-ui/...
```

#### 檢查

回到`pkg/ui/data/swagger`目錄，檢查是否存在`datafile.go`檔案

### `Swagger UI`檔案伺服器（對外提供服務）

在這一步，我們需要使用與其配套的[go-bindata-assetfs](https://github.com/elazarl/go-bindata-assetfs/)

它能夠使用`go-bindata`所生成`Swagger UI`的`Go`程式碼，結合`net/http`對外提供服務

#### 安裝

```
go get github.com/elazarl/go-bindata-assetfs/...
```

#### 編寫

透過分析，我們得知生成的檔案提供了一個`assetFS`函式，該函式返回一個封裝了嵌入檔案的`http.Filesystem`，可以用其來提供一個`HTTP`服務

那麼我們來編寫`Swagger UI`的程式碼吧，主要是兩個部分，一個是`swagger.json`，另外一個是`swagger-ui`的響應

**serveSwaggerFile**

引用包`strings`、`path`

```go
func serveSwaggerFile(w http.ResponseWriter, r *http.Request) {
      if ! strings.HasSuffix(r.URL.Path, "swagger.json") {
        log.Printf("Not Found: %s", r.URL.Path)
        http.NotFound(w, r)
        return
    }

    p := strings.TrimPrefix(r.URL.Path, "/swagger/")
    p = path.Join("proto", p)

    log.Printf("Serving swagger-file: %s", p)

    http.ServeFile(w, r, p)
}
```
在函式中，我們利用`r.URL.Path`進行路徑字尾判斷

主要做了對`swagger.json`的檔案訪問支援（提供`https://127.0.0.1:50052/swagger/hello.swagger.json`的訪問）

**serveSwaggerUI**

引用包`github.com/elazarl/go-bindata-assetfs`、`grpc-hello-world/pkg/ui/data/swagger`

```go
func serveSwaggerUI(mux *http.ServeMux) {
    fileServer := http.FileServer(&assetfs.AssetFS{
        Asset:    swagger.Asset,
        AssetDir: swagger.AssetDir,
        Prefix:   "third_party/swagger-ui",
    })
    prefix := "/swagger-ui/"
    mux.Handle(prefix, http.StripPrefix(prefix, fileServer))
}
```
在函式中，我們使用了[go-bindata-assetfs](https://github.com/elazarl/go-bindata-assetfs/)來排程先前生成的`datafile.go`，結合`net/http`來對外提供`swagger-ui`的服務

#### 結合

在完成功能後，我們發現`path.Join("proto", p)`是寫死引數的，這樣顯然不對，我們應該將其匯出成外部引數，那麼我們來最終改造一番

首先我們在`server.go`新增包全域性變數`SwaggerDir`，修改`cmd/server.go`檔案：

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

        server.Run()
    },
}

func init() {
    serverCmd.Flags().StringVarP(&server.ServerPort, "port", "p", "50052", "server port")
    serverCmd.Flags().StringVarP(&server.CertPemPath, "cert-pem", "", "./conf/certs/server.pem", "cert-pem path")
    serverCmd.Flags().StringVarP(&server.CertKeyPath, "cert-key", "", "./conf/certs/server.key", "cert-key path")
    serverCmd.Flags().StringVarP(&server.CertServerName, "cert-server-name", "", "grpc server name", "server's hostname")
    serverCmd.Flags().StringVarP(&server.SwaggerDir, "swagger-dir", "", "proto", "path to the directory which contains swagger definitions")

    rootCmd.AddCommand(serverCmd)
}
```
修改`path.Join("proto", p)`為`path.Join(SwaggerDir, p)`，這樣的話我們`swagger.json`的檔案路徑就可以根據外部情況去修改它

最終`server.go`檔案內容：

```go
package server

import (
    "crypto/tls"
    "net"
    "net/http"
    "log"
    "strings"
    "path"

    "golang.org/x/net/context"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
    "github.com/grpc-ecosystem/grpc-gateway/runtime"
    "github.com/elazarl/go-bindata-assetfs"

    pb "grpc-hello-world/proto"
    "grpc-hello-world/pkg/util"
    "grpc-hello-world/pkg/ui/data/swagger"
)

var (
    ServerPort string
    CertServerName string
    CertPemPath string
    CertKeyPath string
    SwaggerDir string
    EndPoint string

    tlsConfig *tls.Config
)

func Run() (err error) {
    EndPoint = ":" + ServerPort
    tlsConfig = util.GetTLSConfig(CertPemPath, CertKeyPath)

    conn, err := net.Listen("tcp", EndPoint)
    if err != nil {
        log.Printf("TCP Listen err:%v\n", err)
    }

    srv := newServer(conn)

    log.Printf("gRPC and https listen on: %s\n", ServerPort)

    if err = srv.Serve(util.NewTLSListener(conn, tlsConfig)); err != nil {
        log.Printf("ListenAndServe: %v\n", err)
    }

    return err
}

func newServer(conn net.Listener) (*http.Server) {
    grpcServer := newGrpc()
    gwmux, err := newGateway()
    if err != nil {
        panic(err)
    }

    mux := http.NewServeMux()
    mux.Handle("/", gwmux)
    mux.HandleFunc("/swagger/", serveSwaggerFile)
    serveSwaggerUI(mux)

    return &http.Server{
        Addr:      EndPoint,
        Handler:   util.GrpcHandlerFunc(grpcServer, mux),
        TLSConfig: tlsConfig,
    }
}

func newGrpc() *grpc.Server {
    creds, err := credentials.NewServerTLSFromFile(CertPemPath, CertKeyPath)
    if err != nil {
        panic(err)
    }

    opts := []grpc.ServerOption{
        grpc.Creds(creds),
    }
    server := grpc.NewServer(opts...)

    pb.RegisterHelloWorldServer(server, NewHelloService())

    return server
}

func newGateway() (http.Handler, error) {
    ctx := context.Background()
    dcreds, err := credentials.NewClientTLSFromFile(CertPemPath, CertServerName)
    if err != nil {
        return nil, err
    }
    dopts := []grpc.DialOption{grpc.WithTransportCredentials(dcreds)}

    gwmux := runtime.NewServeMux()
    if err := pb.RegisterHelloWorldHandlerFromEndpoint(ctx, gwmux, EndPoint, dopts); err != nil {
        return nil, err
    }

    return gwmux, nil
}

func serveSwaggerFile(w http.ResponseWriter, r *http.Request) {
      if ! strings.HasSuffix(r.URL.Path, "swagger.json") {
        log.Printf("Not Found: %s", r.URL.Path)
        http.NotFound(w, r)
        return
    }

    p := strings.TrimPrefix(r.URL.Path, "/swagger/")
    p = path.Join(SwaggerDir, p)

    log.Printf("Serving swagger-file: %s", p)

    http.ServeFile(w, r, p)
}

func serveSwaggerUI(mux *http.ServeMux) {
    fileServer := http.FileServer(&assetfs.AssetFS{
        Asset:    swagger.Asset,
        AssetDir: swagger.AssetDir,
        Prefix:   "third_party/swagger-ui",
    })
    prefix := "/swagger-ui/"
    mux.Handle(prefix, http.StripPrefix(prefix, fileServer))
}
```
## 測試

訪問路徑`https://127.0.0.1:50052/swagger/hello.swagger.json`，檢視輸出內容是否為`hello.swagger.json`的內容，例如： \[image]

訪問路徑`https://127.0.0.1:50052/swagger-ui/`，檢視內容 \[image]

## 小結

至此我們這一章節就完畢了，`Swagger`和其生態圈十分的豐富，有興趣研究的小夥伴可以到其[官網](https://swagger.io/)認真研究

而目前完成的程度也滿足了日常工作的需求了，可較自動化的生成`RESTful Api`文件，完成與介面對接

## 參考

### 示例程式碼

* [grpc-hello-world](https://github.com/EDDYCJY/grpc-hello-world)
