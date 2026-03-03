# 3.11 Cron定時任務

專案地址：<https://github.com/EDDYCJY/go-gin-example>

## 知識點

* 完成定時任務的功能

## 本文目標

在實際的應用專案中，定時任務的使用是很常見的。你是否有過 Golang 如何做定時任務的疑問，莫非是輪詢，在本文中我們將結合我們的專案講述 Cron。

## 介紹

我們將使用 [cron](https://github.com/robfig/cron) 這個包，它實作了 cron 規範解析器和任務執行器，簡單來講就是包含了定時任務所需的功能

### Cron 表示式格式

| 欄位名                   | 是否必填 | 允許的值            | 允許的特殊字元    |
| --------------------- | ---- | --------------- | ---------- |
| 秒（Seconds）            | Yes  | 0-59            | \* / , -   |
| 分（Minutes）            | Yes  | 0-59            | \* / , -   |
| 時（Hours）              | Yes  | 0-23            | \* / , -   |
| 一個月中的某天（Day of month） | Yes  | 1-31            | \* / , - ? |
| 月（Month）              | Yes  | 1-12 or JAN-DEC | \* / , -   |
| 星期幾（Day of week）      | Yes  | 0-6 or SUN-SAT  | \* / , - ? |

Cron表示式表示一組時間，使用 6 個空格分隔的欄位

可以留意到 Golang 的 Cron 比 Crontab 多了一個秒級，以後遇到秒級要求的時候就省事了

### Cron 特殊字元

1、星號 ( \* )

星號表示將匹配欄位的所有值

2、斜線 ( / )

斜線使用者 描述範圍的增量，表現為 “N-MAX/x”，first-last/x 的形式，例如 3-59/15 表示此時的第三分鐘和此後的每 15 分鐘，到59分鐘為止。即從 N 開始，使用增量直到該特定範圍結束。它不會重複

3、逗號 ( , )

逗號用於分隔列表中的專案。例如，在 Day of week 使用“MON，WED，FRI”將意味著星期一，星期三和星期五

4、連字元 ( - )

連字元用於定義範圍。例如，9 - 17 表示從上午 9 點到下午 5 點的每個小時

5、問號 ( ? )

不指定值，用於代替 “ \* ”，類似 “ \_ ” 的存在，不難理解

### 預定義的 Cron 時間表

| 輸入                     | 簡述                  | 相當於          |
| ---------------------- | ------------------- | ------------ |
| @yearly (or @annually) | 1月1日午夜執行一次          | 0 0 0 1 1 \* |
| @monthly               | 每個月的午夜，每個月的第一個月執行一次 | 0 0 0 1      |
| @weekly                | 每週一次，週日午夜執行一次       | 0 0 0   0    |
| @daily (or @midnight)  | 每天午夜執行一次            | 0 0 0   \*   |
| @hourly                | 每小時執行一次             | 0 0          |

## 安裝

```
$ go get -u github.com/robfig/cron
```

## 實踐

在上一章節 [Gin實踐 連載十 定製 GORM Callbacks](https://segmentfault.com/a/1190000014393602) 中，我們使用了 GORM 的回撥實作了軟刪除，同時也引入了另外一個問題

就是我怎麼硬刪除，我什麼時候硬刪除？這個往往與業務場景有關係，大致為

* 另外有一套硬刪除介面
* 定時任務清理（或轉移、backup）無效資料

在這裡我們選用第二種解決方案來進行實踐

### 編寫硬刪除程式碼

開啟 models 目錄下的 tag.go、article.go檔案，分別新增以下程式碼

1、tag.go

```go
func CleanAllTag() bool {
    db.Unscoped().Where("deleted_on != ? ", 0).Delete(&Tag{})

    return true
}
```
2、article.go

```go
func CleanAllArticle() bool {
    db.Unscoped().Where("deleted_on != ? ", 0).Delete(&Article{})

    return true
}
```
注意硬刪除要使用 `Unscoped()`，這是 GORM 的約定

### 編寫Cron

在 專案根目錄下新建 cron.go 檔案，用於編寫定時任務的程式碼，寫入檔案內容

```go
package main

import (
    "time"
    "log"

    "github.com/robfig/cron"

    "github.com/EDDYCJY/go-gin-example/models"
)

func main() {
    log.Println("Starting...")

    c := cron.New()
    c.AddFunc("* * * * * *", func() {
        log.Println("Run models.CleanAllTag...")
        models.CleanAllTag()
    })
    c.AddFunc("* * * * * *", func() {
        log.Println("Run models.CleanAllArticle...")
        models.CleanAllArticle()
    })

    c.Start()

    t1 := time.NewTimer(time.Second * 10)
    for {
        select {
        case <-t1.C:
            t1.Reset(time.Second * 10)
        }
    }
}
```
在這段程式中，我們做了如下的事情

#### cron.New()

會根據本地時間建立一個新（空白）的 Cron job runner

```go
func New() *Cron {
    return NewWithLocation(time.Now().Location())
}

// NewWithLocation returns a new Cron job runner.
func NewWithLocation(location *time.Location) *Cron {
    return &Cron{
        entries:  nil,
        add:      make(chan *Entry),
        stop:     make(chan struct{}),
        snapshot: make(chan []*Entry),
        running:  false,
        ErrorLog: nil,
        location: location,
    }
}
```
#### c.AddFunc()

AddFunc 會向 Cron job runner 新增一個 func ，以按給定的時間表執行

```go
func (c *Cron) AddJob(spec string, cmd Job) error {
    schedule, err := Parse(spec)
    if err != nil {
        return err
    }
    c.Schedule(schedule, cmd)
    return nil
}
```
會首先解析時間表，如果填寫有問題會直接 err，無誤則將 func 新增到 Schedule 佇列中等待執行

```go
func (c *Cron) Schedule(schedule Schedule, cmd Job) {
    entry := &Entry{
        Schedule: schedule,
        Job:      cmd,
    }
    if !c.running {
        c.entries = append(c.entries, entry)
        return
    }

    c.add <- entry
}
```
3、c.Start()

在當前執行的程式中啟動 Cron 排程程式。其實這裡的主體是 goroutine + for + select + timer 的排程控制哦

```go
func (c *Cron) Run() {
    if c.running {
        return
    }
    c.running = true
    c.run()
}
```

#### time.NewTimer + for + select + t1.Reset

如果你是初學者，大概會有疑問，這是幹嘛用的？

**（1）time.NewTimer**&#x20;

會建立一個新的定時器，持續你設定的時間 d 後傳送一個 channel 訊息

**（2）for + select**

阻塞 select 等待 channel

**（3）t1.Reset**

會重置定時器，讓它重新開始計時

注：本文適用於 “t.C已經取走，可直接使用 Reset”。

總的來說，這段程式是為了阻塞主程式而編寫的，希望你帶著疑問來想，有沒有別的辦法呢？

有的，你直接 `select{}` 也可以完成這個需求 :)

## 驗證

```
$ go run cron.go 
2018/04/29 17:03:34 [info] replacing callback `gorm:update_time_stamp` from /Users/eddycjy/go/src/github.com/EDDYCJY/go-gin-example/models/models.go:56
2018/04/29 17:03:34 [info] replacing callback `gorm:update_time_stamp` from /Users/eddycjy/go/src/github.com/EDDYCJY/go-gin-example/models/models.go:57
2018/04/29 17:03:34 [info] replacing callback `gorm:delete` from /Users/eddycjy/go/src/github.com/EDDYCJY/go-gin-example/models/models.go:58
2018/04/29 17:03:34 Starting...
2018/04/29 17:03:35 Run models.CleanAllArticle...
2018/04/29 17:03:35 Run models.CleanAllTag...
2018/04/29 17:03:36 Run models.CleanAllArticle...
2018/04/29 17:03:36 Run models.CleanAllTag...
2018/04/29 17:03:37 Run models.CleanAllTag...
2018/04/29 17:03:37 Run models.CleanAllArticle...
```

檢查輸出日誌正常，模擬已軟刪除的資料，定時任務工作OK

## 小結

定時任務很常見，希望你透過本文能夠熟知 Golang 怎麼實作一個簡單的定時任務排程管理

可以不依賴系統的 Crontab 設定，指不定哪一天就用上了呢

## 問題

如果你手動修改計算機的系統時間，是會導致定時任務錯亂的，所以一般不要亂來。

## 參考

### 本系列示例程式碼

* [go-gin-example](https://github.com/EDDYCJY/go-gin-example)

## 關於

### 修改記錄

* 第一版：2018年02月16日釋出文章
* 第二版：2019年10月02日修改文章

## ？

如果有任何疑問或錯誤，歡迎在 [issues](https://github.com/EDDYCJY/blog) 進行提問或給予修正意見，如果喜歡或對你有所幫助，歡迎 Star，對作者是一種鼓勵和推進。

### 我的微信公眾號

![image](../images/3051dc7d3068-8d0b0c3a11e74efd5fdfd7910257e70b.jpg)
