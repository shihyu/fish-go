# 8.3 unsafe

在上一篇文章[《深入理解 Go Slice》](https://book.eddycjy.com/golang/slice/slice.html)中，大家會發現其底層資料結構使用了 `unsafe.Pointer`。因此想著再介紹一下其關聯知識

## 前言

在大家學習 Go 的時候，肯定都學過 “Go 的指標是不支援指標運算和轉換” 這個知識點。為什麼呢？

首先，Go 是一門靜態語言，所有的變數都必須為標量型別。不同的型別不能夠進行賦值、計算等跨型別的操作。那麼指標也對應著相對的型別，也在 Compile 的靜態型別檢查的範圍內。同時靜態語言，也稱為強型別。也就是一旦定義了，就不能再改變它

## 錯誤示例

```go
func main(){
    num := 5
    numPointer := &num

    flnum := (*float32)(numPointer)
    fmt.Println(flnum)
}
```
輸出結果：

```
# command-line-arguments
...: cannot convert numPointer (type *int) to type *float32
```

在示例中，我們建立了一個 `num` 變數，值為 5，型別為 `int`。取了其對於的指標地址後，試圖強制轉換為 `*float32`，結果失敗...

## unsafe

針對剛剛的 “錯誤示例”，我們可以採用今天的男主角 `unsafe` 標準庫來解決。它是一個神奇的包，在官方的詮釋中，有如下概述：

* 圍繞 Go 程式記憶體安全及型別的操作
* 很可能會是不可移植的
* 不受 Go 1 相容性指南的保護

簡單來講就是，不怎麼推薦你使用。因為它是 unsafe（不安全的），但是在特殊的場景下，使用了它。可以打破 Go 的型別和記憶體安全機制，讓你獲得眼前一亮的驚喜效果 😄

### Pointer

為了解決這個問題，需要用到 `unsafe.Pointer`。它表示任意型別且可定址的指標值，可以在不同的指標型別之間進行轉換（類似 C 語言的 void \* 的用途）

其包含四種核心操作：

* 任何型別的指標值都可以轉換為 Pointer
* Pointer 可以轉換為任何型別的指標值
* uintptr 可以轉換為 Pointer
* Pointer 可以轉換為 uintptr

在這一部分，重點看第一點、第二點。你再想想怎麼修改 “錯誤示例” 讓它執行起來？

```go
func main(){
    num := 5
    numPointer := &num

    flnum := (*float32)(unsafe.Pointer(numPointer))
    fmt.Println(flnum)
}
```
輸出結果：

```
0xc4200140b0
```

在上述程式碼中，我們小加改動。透過 `unsafe.Pointer` 的特性對該指標變數進行了修改，就可以完成任意型別（\*T）的指標轉換

需要注意的是，這時還無法對變數進行操作或訪問。因為不知道該指標地址指向的東西具體是什麼型別。不知道是什麼型別，又如何進行解析呢。無法解析也就自然無法對其變更了

### Offsetof

在上小節中，我們對普通的指標變數進行了修改。那麼它是否能做更復雜一點的事呢？

```go
type Num struct{
    i string
    j int64
}

func main(){
    n := Num{i: "EDDYCJY", j: 1}
    nPointer := unsafe.Pointer(&n)

    niPointer := (*string)(unsafe.Pointer(nPointer))
    *niPointer = "煎鱼"

    njPointer := (*int64)(unsafe.Pointer(uintptr(nPointer) + unsafe.Offsetof(n.j)))
    *njPointer = 2

    fmt.Printf("n.i: %s, n.j: %d", n.i, n.j)
}
```
輸出結果：

```
n.i: 煎鱼, n.j: 2
```

在剖析這段程式碼做了什麼事之前，我們需要了解結構體的一些基本概念：

* 結構體的成員變數在記憶體儲存上是一段連續的記憶體
* 結構體的初始地址就是第一個成員變數的記憶體地址
* 基於結構體的成員地址去計算偏移量。就能夠得出其他成員變數的記憶體地址

再回來看看上述程式碼，得出執行流程：

* 修改 `n.i` 值：`i` 為第一個成員變數。因此不需要進行偏移量計算，直接取出指標後轉換為 `Pointer`，再強制轉換為字串型別的指標值即可
* 修改 `n.j` 值：`j` 為第二個成員變數。需要進行偏移量計算，才可以對其記憶體地址進行修改。在進行了偏移運算後，當前地址已經指向第二個成員變數。接著重複轉換賦值即可

需要注意的是，這裡使用瞭如下方法（來完成偏移計算的目標）：

1、uintptr：`uintptr` 是 Go 的內建型別。返回無符號整數，可儲存一個完整的地址。後續常用於指標運算

```
type uintptr uintptr
```

2、unsafe.Offsetof：返回成員變數 x 在結構體當中的偏移量。更具體的講，就是返回結構體初始位置到 x 之間的位元組數。需要注意的是入參 `ArbitraryType` 表示任意型別，並非定義的 `int`。它實際作用是一個佔位符

```go
func Offsetof(x ArbitraryType) uintptr
```
在這一部分，其實就是巧用了 `Pointer` 的第三、第四點特性。這時候就已經可以對變數進行操作了 😄

### 錯誤示例

```go
func main(){
    n := Num{i: "EDDYCJY", j: 1}
    nPointer := unsafe.Pointer(&n)
    ...

    ptr := uintptr(nPointer)
    njPointer := (*int64)(unsafe.Pointer(ptr + unsafe.Offsetof(n.j)))
    ...
}
```
這裡存在一個問題，`uintptr` 型別是不能儲存在臨時變數中的。因為從 GC 的角度來看，`uintptr` 型別的臨時變數只是一個無符號整數，並不知道它是一個指標地址

因此當滿足一定條件後，`ptr` 這個臨時變數是可能被垃圾回收掉的，那麼接下來的記憶體操作，豈不成迷？

## 總結

簡潔回顧兩個知識點。第一是 `unsafe.Pointer` 可以讓你的變數在不同的指標型別轉來轉去，也就是表示為任意可定址的指標型別。第二是 `uintptr` 常用於與 `unsafe.Pointer` 打配合，用於做指標運算，巧妙地很

最後還是那句，沒有特殊必要的話。是不建議使用 `unsafe` 標準庫，它並不安全。雖然它常常能讓你眼前一亮 👌
