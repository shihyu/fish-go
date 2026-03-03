# 3.19 請入門 Makefile

專案地址：<https://github.com/EDDYCJY/go-gin-example>

## 知識點

* 寫一個 Makefile

## 本文目標

含一定複雜度的軟體工程，基本上都是先編譯 A，再依賴 B，再編譯 C...，最後才執行構建。如果每次都人為編排，又或是每新來一個同事就問你專案 D 怎麼構建、重新構建需要注意什麼...等等情況，豈不是要崩潰？

我們常常會在開源專案中發現 Makefile，你是否有過疑問？

本章節會簡單介紹 Makefile 的使用方式，最後建議深入學習。

## 怎麼解決

對於構建編排，Docker 有 Dockerfile ，在 Unix 中有神器 [Make](https://en.wikipedia.org/wiki/Make_\(software\)) ....

## Make

### 是什麼

Make 是一個構建自動化工具，會在當前目錄下尋找 Makefile 或 makefile 檔案。如果存在，會依據 Makefile 的**構建規則**去完成構建

當然了，實際上 Makefile 內都是你根據 make 語法規則，自己編寫的特定 Shell 命令等

它是一個工具，規則也很簡單。在支援的範圍內，編譯 A， 依賴 B，再編譯 C，完全沒問題

### 規則

Makefile 由多條規則組成，每條規則都以一個 target（目標）開頭，後跟一個 : 冒號，冒號後是這一個目標的 prerequisites（前置條件）

緊接著新的一行，必須以一個 tab 作為開頭，後面跟隨 command（命令），也就是你希望這一個 target 所執行的構建命令

```
[target] ... : [prerequisites] ...
<tab>[command]
    ...
    ...
```

* target：一個目標代表一條規則，可以是一個或多個檔名。也可以是某個操作的名字（標籤），稱為**偽目標（phony）**
* prerequisites：前置條件，這一項是**可選引數**。通常是多個檔名、偽目標。它的作用是 target 是否需要重新構建的標準，如果前置條件不存在或有過更新（檔案的最後一次修改時間）則認為 target 需要重新構建
* command：構建這一個 target 的具體命令集

### 簡單的例子

本文將以 [go-gin-example](https://github.com/EDDYCJY/go-gin-example) 去編寫 Makefile 檔案，請跨入 make 的大門

#### 分析

在編寫 Makefile 前，需要先分析構建先後順序、依賴項，需要解決的問題等

#### 編寫

```
.PHONY: build clean tool lint help

all: build

build:
    go build -v .

tool:
    go tool vet . |& grep -v vendor; true
    gofmt -w .

lint:
    golint ./...

clean:
    rm -rf go-gin-example
    go clean -i .

help:
    @echo "make: compile packages and dependencies"
    @echo "make tool: run specified go tool"
    @echo "make lint: golint ./..."
    @echo "make clean: remove object files and cached files"
```

1、在上述檔案中，使用了 `.PHONY`，其作用是宣告 build / clean / tool / lint / help 為**偽目標**，宣告為偽目標會怎麼樣呢？

* 宣告為偽目標後：在執行對應的命令時，make 就不會去檢查是否存在 build / clean / tool / lint / help 其對應的檔案，而是每次都會執行標籤對應的命令
* 若不宣告：恰好存在對應的檔案，則 make 將會認為 xx 檔案已存在，沒有重新構建的必要了

2、這塊比較簡單，在命令列執行即可看見效果，實作了以下功能：

1. make: make 就是 make all
2. make build: 編譯當前專案的包和依賴項
3. make tool: 執行指定的 Go 工具集
4. make lint: golint 一下
5. make clean: 刪除物件檔案和快取檔案
6. make help: help

#### 為什麼會列印執行的命令

如果你實際操作過，可能會有疑問。明明只是執行命令，為什麼會列印到標準輸出上了？

**原因**

make 預設會列印每條命令，再執行。這個行為被定義為**回聲**

**解決**

可以在對應命令前加上 @，可指定該命令不被列印到標準輸出上

```
build:
    @go build -v .
```

那麼還有其他的特殊符號嗎？有的，請課後去了解下 +、- 的用途 🤩

## 小結

這是一篇比較簡潔的文章，希望可以讓您對 Makefile 有一個基本瞭解。

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
