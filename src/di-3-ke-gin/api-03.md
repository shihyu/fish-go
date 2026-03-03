# 3.4 Gin搭建Blog API's （三）

專案地址：<https://github.com/EDDYCJY/go-gin-example>

## 涉及知識點

* [Gin](https://github.com/gin-gonic/gin)：Golang的一個微框架，效能極佳。
* [beego-validation](https://github.com/astaxie/beego/tree/master/validation)：本節採用的beego的表單驗證庫，[中文文件](https://beego.me/docs/mvc/controller/validation.md)。
* [gorm](https://github.com/jinzhu/gorm)，對開發人員友好的ORM框架，[英文文件](http://gorm.io/docs/)
* [com](https://github.com/Unknwon/com)，一個小而美的工具包。

## 本文目標

* 完成部落格的文章類介面定義和編寫

## 定義介面

本節編寫文章的邏輯，我們定義一下介面吧！

* 取得文章列表：GET("/articles")
* 取得指定文章：POST("/articles/:id")
* 新建文章：POST("/articles")
* 更新指定文章：PUT("/articles/:id")
* 刪除指定文章：DELETE("/articles/:id")

## 編寫路由邏輯

在`routers`的v1版本下，新建`article.go`檔案，寫入內容：

```go
package v1

import (
    "github.com/gin-gonic/gin"
)

//获取单个文章
func GetArticle(c *gin.Context) {
}

//获取多个文章
func GetArticles(c *gin.Context) {
}

//新增文章
func AddArticle(c *gin.Context) {
}

//修改文章
func EditArticle(c *gin.Context) {
}

//删除文章
func DeleteArticle(c *gin.Context) {
}
```
我們開啟`routers`下的`router.go`檔案，修改檔案內容為：

```go
package routers

import (
    "github.com/gin-gonic/gin"

    "github.com/EDDYCJY/go-gin-example/routers/api/v1"
    "github.com/EDDYCJY/go-gin-example/pkg/setting"
)

func InitRouter() *gin.Engine {
    ...
    apiv1 := r.Group("/api/v1")
    {
        ...
        //获取文章列表
        apiv1.GET("/articles", v1.GetArticles)
        //获取指定文章
        apiv1.GET("/articles/:id", v1.GetArticle)
        //新建文章
        apiv1.POST("/articles", v1.AddArticle)
        //更新指定文章
        apiv1.PUT("/articles/:id", v1.EditArticle)
        //删除指定文章
        apiv1.DELETE("/articles/:id", v1.DeleteArticle)
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
├── models
│   ├── models.go
│   └── tag.go
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
│   │       ├── article.go
│   │       └── tag.go
│   └── router.go
├── runtime
```

在基礎的路由規則設定結束後，我們**開始編寫我們的介面**吧！

## 編寫models邏輯

建立`models`目錄下的`article.go`，寫入檔案內容：

```go
package models

import (
    "github.com/jinzhu/gorm"

    "time"
)

type Article struct {
    Model

    TagID int `json:"tag_id" gorm:"index"`
    Tag   Tag `json:"tag"`

    Title string `json:"title"`
    Desc string `json:"desc"`
    Content string `json:"content"`
    CreatedBy string `json:"created_by"`
    ModifiedBy string `json:"modified_by"`
    State int `json:"state"`
}


func (article *Article) BeforeCreate(scope *gorm.Scope) error {
    scope.SetColumn("CreatedOn", time.Now().Unix())

    return nil
}

func (article *Article) BeforeUpdate(scope *gorm.Scope) error {
    scope.SetColumn("ModifiedOn", time.Now().Unix())

    return nil
}
```
我們建立了一個`Article struct {}`，與`Tag`不同的是，`Article`多了幾項，如下：

1. `gorm:index`，用於宣告這個欄位為索引，如果你使用了自動遷移功能則會有所影響，在不使用則無影響
2. `Tag`欄位，實際是一個巢狀的`struct`，它利用`TagID`與`Tag`模型相互關聯，在執行查詢的時候，能夠達到`Article`、`Tag`關聯查詢的功能
3. `time.Now().Unix()` 返回當前的時間戳

接下來，請確保已對上一章節的內容通讀且瞭解，由於邏輯偏差不會太遠，我們本節直接編寫這五個介面

開啟`models`目錄下的`article.go`，修改檔案內容：

```go
package models

import (
    "time"

    "github.com/jinzhu/gorm"
)

type Article struct {
    Model

    TagID int `json:"tag_id" gorm:"index"`
    Tag   Tag `json:"tag"`

    Title string `json:"title"`
    Desc string `json:"desc"`
    Content string `json:"content"`
    CreatedBy string `json:"created_by"`
    ModifiedBy string `json:"modified_by"`
    State int `json:"state"`
}


func ExistArticleByID(id int) bool {
    var article Article
    db.Select("id").Where("id = ?", id).First(&article)

    if article.ID > 0 {
        return true
    }

    return false
}

func GetArticleTotal(maps interface {}) (count int){
    db.Model(&Article{}).Where(maps).Count(&count)

    return
}

func GetArticles(pageNum int, pageSize int, maps interface {}) (articles []Article) {
    db.Preload("Tag").Where(maps).Offset(pageNum).Limit(pageSize).Find(&articles)

    return
}

func GetArticle(id int) (article Article) {
    db.Where("id = ?", id).First(&article)
    db.Model(&article).Related(&article.Tag)

    return 
}

func EditArticle(id int, data interface {}) bool {
    db.Model(&Article{}).Where("id = ?", id).Updates(data)

    return true
}

func AddArticle(data map[string]interface {}) bool {
    db.Create(&Article {
        TagID : data["tag_id"].(int),
        Title : data["title"].(string),
        Desc : data["desc"].(string),
        Content : data["content"].(string),
        CreatedBy : data["created_by"].(string),
        State : data["state"].(int),
    })

    return true
}

func DeleteArticle(id int) bool {
    db.Where("id = ?", id).Delete(Article{})

    return true
}

func (article *Article) BeforeCreate(scope *gorm.Scope) error {
    scope.SetColumn("CreatedOn", time.Now().Unix())

    return nil
}

func (article *Article) BeforeUpdate(scope *gorm.Scope) error {
    scope.SetColumn("ModifiedOn", time.Now().Unix())

    return nil
}
```
在這裡，我們拿出三點不同來講，如下：

**1、 我們的`Article`是如何關聯到`Tag`？**

```go
func GetArticle(id int) (article Article) {
    db.Where("id = ?", id).First(&article)
    db.Model(&article).Related(&article.Tag)

    return 
}
```
能夠達到關聯，首先是`gorm`本身做了大量的約定俗成

* `Article`有一個結構體成員是`TagID`，就是外來鍵。`gorm`會透過類名+ID的方式去找到這兩個類之間的關聯關係
* `Article`有一個結構體成員是`Tag`，就是我們巢狀在`Article`裡的`Tag`結構體，我們可以透過`Related`進行關聯查詢

**2、 `Preload`是什麼東西，為什麼查詢可以得出每一項的關聯`Tag`？**

```go
func GetArticles(pageNum int, pageSize int, maps interface {}) (articles []Article) {
    db.Preload("Tag").Where(maps).Offset(pageNum).Limit(pageSize).Find(&articles)

    return
}
```
`Preload`就是一個預載入器，它會執行兩條SQL，分別是`SELECT * FROM blog_articles;`和`SELECT * FROM blog_tag WHERE id IN (1,2,3,4);`，那麼在查詢出結構後，`gorm`內部處理對應的對映邏輯，將其填充到`Article`的`Tag`中，會特別方便，並且避免了迴圈查詢

那麼有沒有別的辦法呢，大致是兩種

* `gorm`的`Join`
* 迴圈`Related`

綜合之下，還是`Preload`更好，如果你有更優的方案，歡迎說一下 :)

**3、 `v.(I)` 是什麼？**

`v`表示一個介面值，`I`表示介面型別。這個實際就是Golang中的**型別斷言**，用於判斷一個介面值的實際型別是否為某個型別，或一個非介面值的型別是否實作了某個介面型別

開啟`routers`目錄下v1版本的`article.go`檔案，修改檔案內容：

```go
package v1

import (
    "net/http"
    "log"

    "github.com/gin-gonic/gin"
    "github.com/astaxie/beego/validation"
    "github.com/unknwon/com"

    "github.com/EDDYCJY/go-gin-example/models"
    "github.com/EDDYCJY/go-gin-example/pkg/e"
    "github.com/EDDYCJY/go-gin-example/pkg/setting"
    "github.com/EDDYCJY/go-gin-example/pkg/util"
)

//获取单个文章
func GetArticle(c *gin.Context) {
    id := com.StrTo(c.Param("id")).MustInt()

    valid := validation.Validation{}
    valid.Min(id, 1, "id").Message("ID必须大于0")

    code := e.INVALID_PARAMS
    var data interface {}
    if ! valid.HasErrors() {
        if models.ExistArticleByID(id) {
            data = models.GetArticle(id)
            code = e.SUCCESS
        } else {
            code = e.ERROR_NOT_EXIST_ARTICLE
        }
    } else {
        for _, err := range valid.Errors {
            log.Printf("err.key: %s, err.message: %s", err.Key, err.Message)
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : data,
    })
}

//获取多个文章
func GetArticles(c *gin.Context) {
    data := make(map[string]interface{})
    maps := make(map[string]interface{})
    valid := validation.Validation{}

    var state int = -1
    if arg := c.Query("state"); arg != "" {
        state = com.StrTo(arg).MustInt()
        maps["state"] = state

        valid.Range(state, 0, 1, "state").Message("状态只允许0或1")
    }

    var tagId int = -1
    if arg := c.Query("tag_id"); arg != "" {
        tagId = com.StrTo(arg).MustInt()
        maps["tag_id"] = tagId

        valid.Min(tagId, 1, "tag_id").Message("标签ID必须大于0")
    } 

    code := e.INVALID_PARAMS
    if ! valid.HasErrors() {
        code = e.SUCCESS

        data["lists"] = models.GetArticles(util.GetPage(c), setting.PageSize, maps)
        data["total"] = models.GetArticleTotal(maps)

    } else {
        for _, err := range valid.Errors {
            log.Printf("err.key: %s, err.message: %s", err.Key, err.Message)
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : data,
    })
}

//新增文章
func AddArticle(c *gin.Context) {
    tagId := com.StrTo(c.Query("tag_id")).MustInt()
    title := c.Query("title")
    desc := c.Query("desc")
    content := c.Query("content")
    createdBy := c.Query("created_by")
    state := com.StrTo(c.DefaultQuery("state", "0")).MustInt()

    valid := validation.Validation{}
    valid.Min(tagId, 1, "tag_id").Message("标签ID必须大于0")
    valid.Required(title, "title").Message("标题不能为空")
    valid.Required(desc, "desc").Message("简述不能为空")
    valid.Required(content, "content").Message("内容不能为空")
    valid.Required(createdBy, "created_by").Message("创建人不能为空")
    valid.Range(state, 0, 1, "state").Message("状态只允许0或1")

    code := e.INVALID_PARAMS
    if ! valid.HasErrors() {
        if models.ExistTagByID(tagId) {
            data := make(map[string]interface {})
            data["tag_id"] = tagId
            data["title"] = title
            data["desc"] = desc
            data["content"] = content
            data["created_by"] = createdBy
            data["state"] = state

            models.AddArticle(data)
            code = e.SUCCESS
        } else {
            code = e.ERROR_NOT_EXIST_TAG
        }
    } else {
        for _, err := range valid.Errors {
            log.Printf("err.key: %s, err.message: %s", err.Key, err.Message)
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : make(map[string]interface{}),
    })
}

//修改文章
func EditArticle(c *gin.Context) {
    valid := validation.Validation{}

    id := com.StrTo(c.Param("id")).MustInt()
    tagId := com.StrTo(c.Query("tag_id")).MustInt()
    title := c.Query("title")
    desc := c.Query("desc")
    content := c.Query("content")
    modifiedBy := c.Query("modified_by")

    var state int = -1
    if arg := c.Query("state"); arg != "" {
        state = com.StrTo(arg).MustInt()
        valid.Range(state, 0, 1, "state").Message("状态只允许0或1")
    }

    valid.Min(id, 1, "id").Message("ID必须大于0")
    valid.MaxSize(title, 100, "title").Message("标题最长为100字符")
    valid.MaxSize(desc, 255, "desc").Message("简述最长为255字符")
    valid.MaxSize(content, 65535, "content").Message("内容最长为65535字符")
    valid.Required(modifiedBy, "modified_by").Message("修改人不能为空")
    valid.MaxSize(modifiedBy, 100, "modified_by").Message("修改人最长为100字符")

    code := e.INVALID_PARAMS
    if ! valid.HasErrors() {
        if models.ExistArticleByID(id) {
            if models.ExistTagByID(tagId) {
                data := make(map[string]interface {})
                if tagId > 0 {
                    data["tag_id"] = tagId
                }
                if title != "" {
                    data["title"] = title
                }
                if desc != "" {
                    data["desc"] = desc
                }
                if content != "" {
                    data["content"] = content
                }

                data["modified_by"] = modifiedBy

                models.EditArticle(id, data)
                code = e.SUCCESS
            } else {
                code = e.ERROR_NOT_EXIST_TAG
            }
        } else {
            code = e.ERROR_NOT_EXIST_ARTICLE
        }
    } else {
        for _, err := range valid.Errors {
            log.Printf("err.key: %s, err.message: %s", err.Key, err.Message)
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : make(map[string]string),
    })
}

//删除文章
func DeleteArticle(c *gin.Context) {
    id := com.StrTo(c.Param("id")).MustInt()

    valid := validation.Validation{}
    valid.Min(id, 1, "id").Message("ID必须大于0")

    code := e.INVALID_PARAMS
    if ! valid.HasErrors() {
        if models.ExistArticleByID(id) {
            models.DeleteArticle(id)
            code = e.SUCCESS
        } else {
            code = e.ERROR_NOT_EXIST_ARTICLE
        }
    } else {
        for _, err := range valid.Errors {
            log.Printf("err.key: %s, err.message: %s", err.Key, err.Message)
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : make(map[string]string),
    })
}
```
當前目錄結構：

```
go-gin-example/
├── conf
│   └── app.ini
├── main.go
├── middleware
├── models
│   ├── article.go
│   ├── models.go
│   └── tag.go
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
│   │       ├── article.go
│   │       └── tag.go
│   └── router.go
├── runtime
```

## 驗證功能

我們重啟服務，執行`go run main.go`，檢查控制檯輸出結果

```
$ go run main.go 
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:    export GIN_MODE=release
 - using code:    gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /api/v1/tags              --> gin-blog/routers/api/v1.GetTags (3 handlers)
[GIN-debug] POST   /api/v1/tags              --> gin-blog/routers/api/v1.AddTag (3 handlers)
[GIN-debug] PUT    /api/v1/tags/:id          --> gin-blog/routers/api/v1.EditTag (3 handlers)
[GIN-debug] DELETE /api/v1/tags/:id          --> gin-blog/routers/api/v1.DeleteTag (3 handlers)
[GIN-debug] GET    /api/v1/articles          --> gin-blog/routers/api/v1.GetArticles (3 handlers)
[GIN-debug] GET    /api/v1/articles/:id      --> gin-blog/routers/api/v1.GetArticle (3 handlers)
[GIN-debug] POST   /api/v1/articles          --> gin-blog/routers/api/v1.AddArticle (3 handlers)
[GIN-debug] PUT    /api/v1/articles/:id      --> gin-blog/routers/api/v1.EditArticle (3 handlers)
[GIN-debug] DELETE /api/v1/articles/:id      --> gin-blog/routers/api/v1.DeleteArticle (3 handlers)
```

使用`Postman`檢驗介面是否正常，在這裡大家可以選用合適的引數傳遞方式，此處為了方便展示我選用了 GET/Param 傳參的方式，而後期會改為 POST。

* POST：<http://127.0.0.1:8000/api/v1/articles?tag_id=1&title=test1&desc=test-desc&content=test-content&created_by=test-created&state=1>
* GET：<http://127.0.0.1:8000/api/v1/articles>
* GET：<http://127.0.0.1:8000/api/v1/articles/1>
* PUT：<http://127.0.0.1:8000/api/v1/articles/1?tag_id=1&title=test-edit1&desc=test-desc-edit&content=test-content-edit&modified_by=test-created-edit&state=0>
* DELETE：<http://127.0.0.1:8000/api/v1/articles/1>

至此，我們的API's編寫就到這裡，下一節我們將介紹另外的一些技巧！

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
