# 3.5 使用JWT進行身份校驗

專案地址：<https://github.com/EDDYCJY/go-gin-example>

## 涉及知識點

* JWT

## 本文目標

在前面幾節中，我們已經基本的完成了API's的編寫，但是，還存在一些非常嚴重的問題，例如，我們現在的API是可以隨意呼叫的，這顯然還不安全全，在本文中我們透過 [jwt-go](https://github.com/dgrijalva/jwt-go) （[GoDoc](https://godoc.org/github.com/dgrijalva/jwt-go)）的方式來簡單解決這個問題。

## 下載依賴包

首先，我們下載 jwt-go的依賴包，如下：

```
go get -u github.com/dgrijalva/jwt-go
```

## 編寫 jwt 工具包

我們需要編寫一個`jwt`的工具包，我們在`pkg`下的`util`目錄新建`jwt.go`，寫入檔案內容：

```go
package util

import (
    "time"

    jwt "github.com/dgrijalva/jwt-go"

    "github.com/EDDYCJY/go-gin-example/pkg/setting"
)

var jwtSecret = []byte(setting.JwtSecret)

type Claims struct {
    Username string `json:"username"`
    Password string `json:"password"`
    jwt.StandardClaims
}

func GenerateToken(username, password string) (string, error) {
    nowTime := time.Now()
    expireTime := nowTime.Add(3 * time.Hour)

    claims := Claims{
        username,
        password,
        jwt.StandardClaims {
            ExpiresAt : expireTime.Unix(),
            Issuer : "gin-blog",
        },
    }

    tokenClaims := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    token, err := tokenClaims.SignedString(jwtSecret)

    return token, err
}

func ParseToken(token string) (*Claims, error) {
    tokenClaims, err := jwt.ParseWithClaims(token, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return jwtSecret, nil
    })

    if tokenClaims != nil {
        if claims, ok := tokenClaims.Claims.(*Claims); ok && tokenClaims.Valid {
            return claims, nil
        }
    }

    return nil, err
}
```
在這個工具包，我們涉及到

* `NewWithClaims(method SigningMethod, claims Claims)`，`method`對應著`SigningMethodHMAC  struct{}`，其包含`SigningMethodHS256`、`SigningMethodHS384`、`SigningMethodHS512`三種`crypto.Hash`方案
* `func (t *Token) SignedString(key interface{})` 該方法內部生成簽名字串，再用於取得完整、已簽名的`token`
* `func (p *Parser) ParseWithClaims` 用於解析鑑權的宣告，[方法內部](https://gowalker.org/github.com/dgrijalva/jwt-go#Parser_ParseWithClaims)主要是具體的解碼和校驗的過程，最終返回`*Token`
* `func (m MapClaims) Valid()` 驗證基於時間的宣告`exp, iat, nbf`，注意如果沒有任何宣告在令牌中，仍然會被認為是有效的。並且對於時區偏差沒有計算方法

有了`jwt`工具包，接下來我們要編寫要用於`Gin`的中介軟體，我們在`middleware`下新建`jwt`目錄，新建`jwt.go`檔案，寫入內容：

```go
package jwt

import (
    "time"
    "net/http"

    "github.com/gin-gonic/gin"

    "github.com/EDDYCJY/go-gin-example/pkg/util"
    "github.com/EDDYCJY/go-gin-example/pkg/e"
)

func JWT() gin.HandlerFunc {
    return func(c *gin.Context) {
        var code int
        var data interface{}

        code = e.SUCCESS
        token := c.Query("token")
        if token == "" {
            code = e.INVALID_PARAMS
        } else {
            claims, err := util.ParseToken(token)
            if err != nil {
                code = e.ERROR_AUTH_CHECK_TOKEN_FAIL
            } else if time.Now().Unix() > claims.ExpiresAt {
                code = e.ERROR_AUTH_CHECK_TOKEN_TIMEOUT
            }
        }

        if code != e.SUCCESS {
            c.JSON(http.StatusUnauthorized, gin.H{
                "code" : code,
                "msg" : e.GetMsg(code),
                "data" : data,
            })

            c.Abort()
            return
        }

        c.Next()
    }
}
```
## 如何取得`Token`

那麼我們如何呼叫它呢，我們還要取得`Token`呢？

1、 我們要新增一個取得`Token`的API

在`models`下新建`auth.go`檔案，寫入內容：

```go
package models

type Auth struct {
    ID int `gorm:"primary_key" json:"id"`
    Username string `json:"username"`
    Password string `json:"password"`
}

func CheckAuth(username, password string) bool {
    var auth Auth
    db.Select("id").Where(Auth{Username : username, Password : password}).First(&auth)
    if auth.ID > 0 {
        return true
    }

    return false
}
```
在`routers`下的`api`目錄新建`auth.go`檔案，寫入內容：

```go
package api

import (
    "log"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/astaxie/beego/validation"

    "github.com/EDDYCJY/go-gin-example/pkg/e"
    "github.com/EDDYCJY/go-gin-example/pkg/util"
    "github.com/EDDYCJY/go-gin-example/models"
)

type auth struct {
    Username string `valid:"Required; MaxSize(50)"`
    Password string `valid:"Required; MaxSize(50)"`
}

func GetAuth(c *gin.Context) {
    username := c.Query("username")
    password := c.Query("password")

    valid := validation.Validation{}
    a := auth{Username: username, Password: password}
    ok, _ := valid.Valid(&a)

    data := make(map[string]interface{})
    code := e.INVALID_PARAMS
    if ok {
        isExist := models.CheckAuth(username, password)
        if isExist {
            token, err := util.GenerateToken(username, password)
            if err != nil {
                code = e.ERROR_AUTH_TOKEN
            } else {
                data["token"] = token

                code = e.SUCCESS
            }

        } else {
            code = e.ERROR_AUTH
        }
    } else {
        for _, err := range valid.Errors {
            log.Println(err.Key, err.Message)
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : data,
    })
}
```
我們開啟`routers`目錄下的`router.go`檔案，修改檔案內容（新增取得token的方法）：

```go
package routers

import (
    "github.com/gin-gonic/gin"

    "github.com/EDDYCJY/go-gin-example/routers/api"
    "github.com/EDDYCJY/go-gin-example/routers/api/v1"
    "github.com/EDDYCJY/go-gin-example/pkg/setting"
)

func InitRouter() *gin.Engine {
    r := gin.New()

    r.Use(gin.Logger())

    r.Use(gin.Recovery())

    gin.SetMode(setting.RunMode)

    r.GET("/auth", api.GetAuth)

    apiv1 := r.Group("/api/v1")
    {
        ...
    }

    return r
}
```
## 驗證`Token`

取得`token`的API方法就到這裡啦，讓我們來測試下是否可以正常使用吧！

重啟服務後，用`GET`方式訪問`http://127.0.0.1:8000/auth?username=test&password=test123456`，檢視返回值是否正確

```
{
  "code": 200,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6InRlc3QiLCJwYXNzd29yZCI6InRlc3QxMjM0NTYiLCJleHAiOjE1MTg3MjAwMzcsImlzcyI6Imdpbi1ibG9nIn0.-kK0V9E06qTHOzupQM_gHXAGDB3EJtJS4H5TTCyWwW8"
  },
  "msg": "ok"
}
```

我們有了`token`的API，也呼叫成功了

## 將中介軟體接入`Gin`

2、 接下來我們將中介軟體接入到`Gin`的訪問流程中

我們開啟`routers`目錄下的`router.go`檔案，修改檔案內容（新增引用包和中介軟體引用）

```go
package routers

import (
    "github.com/gin-gonic/gin"

    "github.com/EDDYCJY/go-gin-example/routers/api"
    "github.com/EDDYCJY/go-gin-example/routers/api/v1"
    "github.com/EDDYCJY/go-gin-example/pkg/setting"
    "github.com/EDDYCJY/go-gin-example/middleware/jwt"
)

func InitRouter() *gin.Engine {
    r := gin.New()

    r.Use(gin.Logger())

    r.Use(gin.Recovery())

    gin.SetMode(setting.RunMode)

    r.GET("/auth", api.GetAuth)

    apiv1 := r.Group("/api/v1")
    apiv1.Use(jwt.JWT())
    {
        ...
    }

    return r
}
```
當前目錄結構：

```
go-gin-example/
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

到這裡，我們的`JWT`編寫就完成啦！

## 驗證功能

我們來測試一下，再次訪問

* <http://127.0.0.1:8000/api/v1/articles>
* <http://127.0.0.1:8000/api/v1/articles?token=23131>

正確的反饋應該是

```
{
  "code": 400,
  "data": null,
  "msg": "请求参数错误"
}

{
  "code": 20001,
  "data": null,
  "msg": "Token鉴权失败"
}
```

我們需要訪問`http://127.0.0.1:8000/auth?username=test&password=test123456`，得到`token`

```
{
  "code": 200,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6InRlc3QiLCJwYXNzd29yZCI6InRlc3QxMjM0NTYiLCJleHAiOjE1MTg3MjQ2OTMsImlzcyI6Imdpbi1ibG9nIn0.KSBY6TeavV_30kfmP7HWLRYKP5TPEDgHtABe9HCsic4"
  },
  "msg": "ok"
}
```

再用包含`token`的URL引數去訪問我們的應用API，

訪問`http://127.0.0.1:8000/api/v1/articles?token=eyJhbGci...`，檢查介面返回值

```
{
  "code": 200,
  "data": {
    "lists": [
      {
        "id": 2,
        "created_on": 1518700920,
        "modified_on": 0,
        "tag_id": 1,
        "tag": {
          "id": 1,
          "created_on": 1518684200,
          "modified_on": 0,
          "name": "tag1",
          "created_by": "",
          "modified_by": "",
          "state": 0
        },
        "content": "test-content",
        "created_by": "test-created",
        "modified_by": "",
        "state": 0
      }
    ],
    "total": 1
  },
  "msg": "ok"
}
```

返回正確，至此我們的`jwt-go`在`Gin`中的驗證就完成了！

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
