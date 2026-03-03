# 3.3 Gin搭建Blog API's （二）

專案地址：<https://github.com/EDDYCJY/go-gin-example>

## 涉及知識點

* [Gin](https://github.com/gin-gonic/gin)：Golang的一個微框架，效能極佳。
* [beego-validation](https://github.com/astaxie/beego/tree/master/validation)：本節採用的beego的表單驗證庫，[中文文件](https://beego.me/docs/mvc/controller/validation.md)。
* [gorm](https://github.com/jinzhu/gorm)，對開發人員友好的ORM框架，[英文文件](http://gorm.io/docs/)
* [com](https://github.com/Unknwon/com)，一個小而美的工具包。

## 本文目標

* 完成部落格的標籤類介面定義和編寫

## 定義介面

本節正是編寫標籤的邏輯，我們想一想，一般介面為增刪改查是基礎的，那麼我們定義一下介面吧！

* 取得標籤列表：GET("/tags")
* 新建標籤：POST("/tags")
* 更新指定標籤：PUT("/tags/:id")
* 刪除指定標籤：DELETE("/tags/:id")

## 編寫路由空殼

開始編寫路由檔案邏輯，在`routers`下新建`api`目錄，我們當前是第一個API大版本，因此在`api`下新建`v1`目錄，再新建`tag.go`檔案，寫入內容：

```go
package v1

import (
    "github.com/gin-gonic/gin"
)

//获取多个文章标签
func GetTags(c *gin.Context) {
}

//新增文章标签
func AddTag(c *gin.Context) {
}

//修改文章标签
func EditTag(c *gin.Context) {
}

//删除文章标签
func DeleteTag(c *gin.Context) {
}
```
## 註冊路由

我們開啟`routers`下的`router.go`檔案，修改檔案內容為：

```go
package routers

import (
    "github.com/gin-gonic/gin"

    "gin-blog/routers/api/v1"
    "gin-blog/pkg/setting"
)

func InitRouter() *gin.Engine {
    r := gin.New()

    r.Use(gin.Logger())

    r.Use(gin.Recovery())

    gin.SetMode(setting.RunMode)

    apiv1 := r.Group("/api/v1")
    {
        //获取标签列表
        apiv1.GET("/tags", v1.GetTags)
        //新建标签
        apiv1.POST("/tags", v1.AddTag)
        //更新指定标签
        apiv1.PUT("/tags/:id", v1.EditTag)
        //删除指定标签
        apiv1.DELETE("/tags/:id", v1.DeleteTag)
    }

    return r
}
```
當前目錄結構：

```
gin-blog/
├── conf
│   └── app.ini
├── main.go
├── middleware
├── models
│   └── models.go
├── pkg
│   ├── e
│   │   ├── code.go
│   │   └── msg.go
│   ├── setting
│   │   └── setting.go
│   └── util
│       └── pagination.go
├── routers
│   ├── api
│   │   └── v1
│   │       └── tag.go
│   └── router.go
├── runtime
```

## 檢驗路由是否註冊成功

回到命令列，執行`go run main.go`，檢查路由規則是否註冊成功。

```
$ go run main.go 
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:    export GIN_MODE=release
 - using code:    gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /api/v1/tags              --> gin-blog/routers/api/v1.GetTags (3 handlers)
[GIN-debug] POST   /api/v1/tags              --> gin-blog/routers/api/v1.AddTag (3 handlers)
[GIN-debug] PUT    /api/v1/tags/:id          --> gin-blog/routers/api/v1.EditTag (3 handlers)
[GIN-debug] DELETE /api/v1/tags/:id          --> gin-blog/routers/api/v1.DeleteTag (3 handlers)
```

執行成功，那麼我們愉快的**開始編寫我們的介面**吧！

## 下載依賴包

首先我們要拉取`validation`的依賴包，在後面的接口裡會使用到表單驗證

```
go get -u github.com/astaxie/beego/validation
```

## 編寫標籤列表的models邏輯

建立`models`目錄下的`tag.go`，寫入檔案內容：

```go
package models

type Tag struct {
    Model

    Name string `json:"name"`
    CreatedBy string `json:"created_by"`
    ModifiedBy string `json:"modified_by"`
    State int `json:"state"`
}

func GetTags(pageNum int, pageSize int, maps interface {}) (tags []Tag) {
    db.Where(maps).Offset(pageNum).Limit(pageSize).Find(&tags)

    return
}

func GetTagTotal(maps interface {}) (count int){
    db.Model(&Tag{}).Where(maps).Count(&count)

    return
}
```
1. 我們建立了一個`Tag struct{}`，用於`Gorm`的使用。並給予了附屬屬性`json`，這樣子在`c.JSON`的時候就會自動轉換格式，非常的便利
2. 可能會有的初學者看到`return`，而後面沒有跟著變數，會不理解；其實你可以看到在函式末端，我們已經顯示聲明瞭返回值，這個變數在函式體內也可以直接使用，因為他在一開始就被聲明瞭
3. 有人會疑惑`db`是哪裡來的；因為在同個`models`包下，因此`db *gorm.DB`是可以直接使用的

## 編寫標籤列表的路由邏輯

開啟`routers`目錄下v1版本的`tag.go`，第一我們先編寫**取得標籤列表的介面**

修改檔案內容：

```go
package v1

import (
    "net/http"

    "github.com/gin-gonic/gin"
    //"github.com/astaxie/beego/validation"
    "github.com/Unknwon/com"

    "gin-blog/pkg/e"
    "gin-blog/models"
    "gin-blog/pkg/util"
    "gin-blog/pkg/setting"
)

//获取多个文章标签
func GetTags(c *gin.Context) {
    name := c.Query("name")

    maps := make(map[string]interface{})
    data := make(map[string]interface{})

    if name != "" {
        maps["name"] = name
    }

    var state int = -1
    if arg := c.Query("state"); arg != "" {
        state = com.StrTo(arg).MustInt()
        maps["state"] = state
    }

    code := e.SUCCESS

    data["lists"] = models.GetTags(util.GetPage(c), setting.PageSize, maps)
    data["total"] = models.GetTagTotal(maps)

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : data,
    })
}

//新增文章标签
func AddTag(c *gin.Context) {
}

//修改文章标签
func EditTag(c *gin.Context) {
}

//删除文章标签
func DeleteTag(c *gin.Context) {
}
```
1. `c.Query`可用於取得`?name=test&state=1`這類URL引數，而`c.DefaultQuery`則支援設定一個預設值
2. `code`變數使用了`e`模組的錯誤編碼，這正是先前規劃好的錯誤碼，方便排錯和識別記錄
3. `util.GetPage`保證了各介面的`page`處理是一致的
4. `c *gin.Context`是`Gin`很重要的組成部分，可以理解為上下文，它允許我們在中介軟體之間傳遞變數、管理流、驗證請求的JSON和呈現JSON響應

在本機執行`curl 127.0.0.1:8000/api/v1/tags`，正確的返回值為`{"code":200,"data":{"lists":[],"total":0},"msg":"ok"}`，若存在問題請結合gin結果進行拍錯。

在取得標籤列表介面中，我們可以根據`name`、`state`、`page`來篩選查詢條件，分頁的步長可透過`app.ini`進行設定，以`lists`、`total`的組合返回達到分頁效果。

## 編寫新增標籤的models邏輯

接下來我們編寫**新增標籤**的介面

開啟`models`目錄下的`tag.go`，修改檔案（增加2個方法）：

```go
...
func ExistTagByName(name string) bool {
    var tag Tag
    db.Select("id").Where("name = ?", name).First(&tag)
    if tag.ID > 0 {
        return true
    }

    return false
}

func AddTag(name string, state int, createdBy string) bool{
    db.Create(&Tag {
        Name : name,
        State : state,
        CreatedBy : createdBy,
    })

    return true
}
...
```
## 編寫新增標籤的路由邏輯

開啟`routers`目錄下的`tag.go`，修改檔案（變動AddTag方法）：

```go
package v1

import (
    "log"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/astaxie/beego/validation"
    "github.com/Unknwon/com"

    "gin-blog/pkg/e"
    "gin-blog/models"
    "gin-blog/pkg/util"
    "gin-blog/pkg/setting"
)
...
//新增文章标签
func AddTag(c *gin.Context) {
    name := c.Query("name")
    state := com.StrTo(c.DefaultQuery("state", "0")).MustInt()
    createdBy := c.Query("created_by")

    valid := validation.Validation{}
    valid.Required(name, "name").Message("名称不能为空")
    valid.MaxSize(name, 100, "name").Message("名称最长为100字符")
    valid.Required(createdBy, "created_by").Message("创建人不能为空")
    valid.MaxSize(createdBy, 100, "created_by").Message("创建人最长为100字符")
    valid.Range(state, 0, 1, "state").Message("状态只允许0或1")

    code := e.INVALID_PARAMS
    if ! valid.HasErrors() {
        if ! models.ExistTagByName(name) {
            code = e.SUCCESS
            models.AddTag(name, state, createdBy)
        } else {
            code = e.ERROR_EXIST_TAG
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : make(map[string]string),
    })
}
...
```
用`Postman`用POST訪問`http://127.0.0.1:8000/api/v1/tags?name=1&state=1&created_by=test`，檢視`code`是否返回`200`及`blog_tag`表中是否有值，有值則正確。

## 編寫models callbacks

但是這個時候大家會發現，我明明新增了標籤，但`created_on`居然沒有值，那做修改標籤的時候`modified_on`會不會也存在這個問題？

為了解決這個問題，我們需要開啟`models`目錄下的`tag.go`檔案，修改檔案內容（修改包引用和增加2個方法）：

```go
package models

import (
    "time"

    "github.com/jinzhu/gorm"
)

...

func (tag *Tag) BeforeCreate(scope *gorm.Scope) error {
    scope.SetColumn("CreatedOn", time.Now().Unix())

    return nil
}

func (tag *Tag) BeforeUpdate(scope *gorm.Scope) error {
    scope.SetColumn("ModifiedOn", time.Now().Unix())

    return nil
}
```
重啟服務，再在用`Postman`用POST訪問`http://127.0.0.1:8000/api/v1/tags?name=2&state=1&created_by=test`，發現`created_on`已經有值了！

**在這幾段程式碼中，涉及到知識點：**

這屬於`gorm`的`Callbacks`，可以將回調方法定義為模型結構的指標，在建立、更新、查詢、刪除時將被呼叫，如果任何回撥返回錯誤，gorm將停止未來操作並回滾所有更改。

`gorm`所支援的回撥方法：

* 建立：BeforeSave、BeforeCreate、AfterCreate、AfterSave
* 更新：BeforeSave、BeforeUpdate、AfterUpdate、AfterSave
* 刪除：BeforeDelete、AfterDelete
* 查詢：AfterFind

## 編寫其餘介面的路由邏輯

接下來，我們一口氣把剩餘的兩個介面（EditTag、DeleteTag）完成吧

開啟`routers`目錄下v1版本的`tag.go`檔案，修改內容：

```go
...
//修改文章标签
func EditTag(c *gin.Context) {
    id := com.StrTo(c.Param("id")).MustInt()
    name := c.Query("name")
    modifiedBy := c.Query("modified_by")

    valid := validation.Validation{}

    var state int = -1
    if arg := c.Query("state"); arg != "" {
        state = com.StrTo(arg).MustInt()
        valid.Range(state, 0, 1, "state").Message("状态只允许0或1")
    }

    valid.Required(id, "id").Message("ID不能为空")
    valid.Required(modifiedBy, "modified_by").Message("修改人不能为空")
    valid.MaxSize(modifiedBy, 100, "modified_by").Message("修改人最长为100字符")
    valid.MaxSize(name, 100, "name").Message("名称最长为100字符")

    code := e.INVALID_PARAMS
    if ! valid.HasErrors() {
        code = e.SUCCESS
        if models.ExistTagByID(id) {
            data := make(map[string]interface{})
            data["modified_by"] = modifiedBy
            if name != "" {
                data["name"] = name
            }
            if state != -1 {
                data["state"] = state
            }

            models.EditTag(id, data)
        } else {
            code = e.ERROR_NOT_EXIST_TAG
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : make(map[string]string),
    })
}    

//删除文章标签
func DeleteTag(c *gin.Context) {
    id := com.StrTo(c.Param("id")).MustInt()

    valid := validation.Validation{}
    valid.Min(id, 1, "id").Message("ID必须大于0")

    code := e.INVALID_PARAMS
    if ! valid.HasErrors() {
        code = e.SUCCESS
        if models.ExistTagByID(id) {
            models.DeleteTag(id)
        } else {
            code = e.ERROR_NOT_EXIST_TAG
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : make(map[string]string),
    })
}
```
## 編寫其餘介面的models邏輯

開啟`models`下的`tag.go`，修改檔案內容：

```go
...

func ExistTagByID(id int) bool {
    var tag Tag
    db.Select("id").Where("id = ?", id).First(&tag)
    if tag.ID > 0 {
        return true
    }

    return false
}

func DeleteTag(id int) bool {
    db.Where("id = ?", id).Delete(&Tag{})

    return true
}

func EditTag(id int, data interface {}) bool {
    db.Model(&Tag{}).Where("id = ?", id).Updates(data)

    return true
}
...
```
## 驗證功能

重啟服務，用Postman

* PUT訪問[http://127.0.0.1:8000/api/v1/tags/1?name=edit1\&state=0\&modified\_by=edit1，檢視code是否返回200](http://127.0.0.1:8000/api/v1/tags/1?name=edit1\&state=0\&modified_by=edit1%EF%BC%8C%E6%9F%A5%E7%9C%8Bcode%E6%98%AF%E5%90%A6%E8%BF%94%E5%9B%9E200)
* DELETE訪問[http://127.0.0.1:8000/api/v1/tags/1，檢視code是否返回200](http://127.0.0.1:8000/api/v1/tags/1%EF%BC%8C%E6%9F%A5%E7%9C%8Bcode%E6%98%AF%E5%90%A6%E8%BF%94%E5%9B%9E200)

至此，Tag的API's完成，下一節我們將開始Article的API's編寫！

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
