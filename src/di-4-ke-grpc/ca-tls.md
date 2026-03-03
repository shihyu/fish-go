# 4.5 基於 CA 的 TLS 證書認證

專案地址：<https://github.com/EDDYCJY/go-grpc-example>

## 前言

在上一章節中，我們提出了一個問題。就是如何保證證書的可靠性和有效性？你如何確定你 Server、Client 的證書是對的呢？

## CA

為了保證證書的可靠性和有效性，在這裡可引入 CA 頒發的根證書的概念。其遵守 X.509 標準

### 根證書

根證書（root certificate）是屬於根證書頒發機構（CA）的公鑰證書。我們可以透過驗證 CA 的簽名從而信任 CA ，任何人都可以得到 CA 的證書（含公鑰），用以驗證它所簽發的證書（客戶端、服務端）

它包含的檔案如下：

* 公鑰
* 金鑰

### 生成 Key

```
openssl genrsa -out ca.key 2048
```

### 生成金鑰

```
openssl req -new -x509 -days 7200 -key ca.key -out ca.pem
```

#### 填寫資訊

```
Country Name (2 letter code) []:
State or Province Name (full name) []:
Locality Name (eg, city) []:
Organization Name (eg, company) []:
Organizational Unit Name (eg, section) []:
Common Name (eg, fully qualified host name) []:go-grpc-example
Email Address []:
```

### Server

#### 生成 CSR

```
openssl req -new -key server.key -out server.csr
```

**填寫資訊**

```
Country Name (2 letter code) []:
State or Province Name (full name) []:
Locality Name (eg, city) []:
Organization Name (eg, company) []:
Organizational Unit Name (eg, section) []:
Common Name (eg, fully qualified host name) []:go-grpc-example
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
```

CSR 是 Cerificate Signing Request 的英文縮寫，為證書請求檔案。主要作用是 CA 會利用 CSR 檔案進行簽名使得攻擊者無法偽裝或篡改原有證書

#### 基於 CA 簽發

```
openssl x509 -req -sha256 -CA ca.pem -CAkey ca.key -CAcreateserial -days 3650 -in server.csr -out server.pem
```

### Client

### 生成 Key

```
openssl ecparam -genkey -name secp384r1 -out client.key
```

### 生成 CSR

```
openssl req -new -key client.key -out client.csr
```

#### 基於 CA 簽發

```
openssl x509 -req -sha256 -CA ca.pem -CAkey ca.key -CAcreateserial -days 3650 -in client.csr -out client.pem
```

### 整理目錄

至此我們生成了一堆檔案，請按照以下目錄結構存放：

```
$ tree conf 
conf
├── ca.key
├── ca.pem
├── ca.srl
├── client
│   ├── client.csr
│   ├── client.key
│   └── client.pem
└── server
    ├── server.csr
    ├── server.key
    └── server.pem
```

另外有一些檔案是不應該出現在倉庫內，應當保密或刪除的。但為了真實演示所以保留著（敲黑板）

## gRPC

接下來將正式開始針對 gRPC 進行編碼，改造上一章節的程式碼。目標是基於 CA 進行 TLS 認證 🤫

### Server

```go
package main

import (
    "context"
    "log"
    "net"
    "crypto/tls"
    "crypto/x509"
    "io/ioutil"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"

    pb "github.com/EDDYCJY/go-grpc-example/proto"
)

...

const PORT = "9001"

func main() {
    cert, err := tls.LoadX509KeyPair("../../conf/server/server.pem", "../../conf/server/server.key")
    if err != nil {
        log.Fatalf("tls.LoadX509KeyPair err: %v", err)
    }

    certPool := x509.NewCertPool()
    ca, err := ioutil.ReadFile("../../conf/ca.pem")
    if err != nil {
        log.Fatalf("ioutil.ReadFile err: %v", err)
    }

    if ok := certPool.AppendCertsFromPEM(ca); !ok {
        log.Fatalf("certPool.AppendCertsFromPEM err")
    }

    c := credentials.NewTLS(&tls.Config{
        Certificates: []tls.Certificate{cert},
        ClientAuth:   tls.RequireAndVerifyClientCert,
        ClientCAs:    certPool,
    })

    server := grpc.NewServer(grpc.Creds(c))
    pb.RegisterSearchServiceServer(server, &SearchService{})

    lis, err := net.Listen("tcp", ":"+PORT)
    if err != nil {
        log.Fatalf("net.Listen err: %v", err)
    }

    server.Serve(lis)
}
```
* tls.LoadX509KeyPair()：從證書相關檔案中**讀取**和**解析**資訊，得到證書公鑰、金鑰對

```go
func LoadX509KeyPair(certFile, keyFile string) (Certificate, error) {
    certPEMBlock, err := ioutil.ReadFile(certFile)
    if err != nil {
        return Certificate{}, err
    }
    keyPEMBlock, err := ioutil.ReadFile(keyFile)
    if err != nil {
        return Certificate{}, err
    }
    return X509KeyPair(certPEMBlock, keyPEMBlock)
}
```
* x509.NewCertPool()：建立一個新的、空的 CertPool
* certPool.AppendCertsFromPEM()：嘗試解析所傳入的 PEM 編碼的證書。如果解析成功會將其加到 CertPool 中，便於後面的使用
* credentials.NewTLS：構建基於 TLS 的 TransportCredentials 選項
* tls.Config：Config 結構用於設定 TLS 客戶端或伺服器

在 Server，共使用了三個 Config 設定項：

（1）Certificates：設定證書鏈，允許包含一個或多個

（2）ClientAuth：要求必須校驗客戶端的證書。可以根據實際情況選用以下引數：

```
const (
    NoClientCert ClientAuthType = iota
    RequestClientCert
    RequireAnyClientCert
    VerifyClientCertIfGiven
    RequireAndVerifyClientCert
)
```

（3）ClientCAs：設定根證書的集合，校驗方式使用 ClientAuth 中設定的模式

### Client

```go
package main

import (
    "context"
    "crypto/tls"
    "crypto/x509"
    "io/ioutil"
    "log"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"

    pb "github.com/EDDYCJY/go-grpc-example/proto"
)

const PORT = "9001"

func main() {
    cert, err := tls.LoadX509KeyPair("../../conf/client/client.pem", "../../conf/client/client.key")
    if err != nil {
        log.Fatalf("tls.LoadX509KeyPair err: %v", err)
    }

    certPool := x509.NewCertPool()
    ca, err := ioutil.ReadFile("../../conf/ca.pem")
    if err != nil {
        log.Fatalf("ioutil.ReadFile err: %v", err)
    }

    if ok := certPool.AppendCertsFromPEM(ca); !ok {
        log.Fatalf("certPool.AppendCertsFromPEM err")
    }

    c := credentials.NewTLS(&tls.Config{
        Certificates: []tls.Certificate{cert},
        ServerName:   "go-grpc-example",
        RootCAs:      certPool,
    })

    conn, err := grpc.Dial(":"+PORT, grpc.WithTransportCredentials(c))
    if err != nil {
        log.Fatalf("grpc.Dial err: %v", err)
    }
    defer conn.Close()

    client := pb.NewSearchServiceClient(conn)
    resp, err := client.Search(context.Background(), &pb.SearchRequest{
        Request: "gRPC",
    })
    if err != nil {
        log.Fatalf("client.Search err: %v", err)
    }

    log.Printf("resp: %s", resp.GetResponse())
}
```
在 Client 中絕大部分與 Server 一致，不同點的地方是，在 Client 請求 Server 端時，Client 端會使用根證書和 ServerName 去對 Server 端進行校驗

簡單流程大致如下：

1. Client 透過請求得到 Server 端的證書
2. 使用 CA 認證的根證書對 Server 端的證書進行可靠性、有效性等校驗
3. 校驗 ServerName 是否可用、有效

當然了，在設定了 `tls.RequireAndVerifyClientCert` 模式的情況下，Server 也會使用 CA 認證的根證書對 Client 端的證書進行可靠性、有效性等校驗。也就是兩邊都會進行校驗，極大的保證了安全性 👍

### 驗證

重新啟動 server.go 和執行 client.go，檢視響應結果是否正常

## 總結

在本章節，我們使用 CA 頒發的根證書對客戶端、服務端的證書進行了簽發。進一步的提高了兩者的通訊安全

這回是真的大功告成了！

## 參考

### 本系列示例程式碼

* [go-grpc-example](https://github.com/EDDYCJY/go-grpc-example)
