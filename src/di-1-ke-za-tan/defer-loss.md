# 1.10 defer 會有效能損耗，儘量不要用

![image](../images/f4e011af03ac-YlKjnSH.jpg)

上個月在 @polaris @軒脈刃 的全棧技術群裡看到一個小夥伴問 **“說 defer 在棧退出時執行，會有效能損耗，儘量不要用，這個怎麼解？”**。

恰好前段時間寫了一篇 [《深入理解 Go defer》](https://segmentfault.com/a/1190000019303572) 去詳細剖析 `defer` 關鍵字。那麼這一次簡單結合前文對這個問題進行探討一波，希望對你有所幫助，但在此之前希望你花幾分鐘，自己思考一下答案，再繼續往下看。

## 測試

```go
func DoDefer(key, value string) {
    defer func(key, value string) {
        _ = key + value
    }(key, value)
}

func DoNotDefer(key, value string) {
    _ = key + value
}
```
基準測試：

```go
func BenchmarkDoDefer(b *testing.B) {
    for i := 0; i < b.N; i++ {
        DoDefer("煎鱼", "https://github.com/EDDYCJY/blog")
    }
}

func BenchmarkDoNotDefer(b *testing.B) {
    for i := 0; i < b.N; i++ {
        DoNotDefer("煎鱼", "https://github.com/EDDYCJY/blog")
    }
}
```
輸出結果：

```
$ go test -bench=. -benchmem -run=none
goos: darwin
goarch: amd64
pkg: github.com/EDDYCJY/awesomeDefer
BenchmarkDoDefer-4          20000000            91.4 ns/op          48 B/op           1 allocs/op
BenchmarkDoNotDefer-4       30000000            41.6 ns/op          48 B/op           1 allocs/op
PASS
ok      github.com/EDDYCJY/awesomeDefer    3.234s
```

從結果上來，使用 `defer` 後的函式開銷確實比沒使用高了不少，這損耗用到哪裡去了呢？

## 想一下

```
$ go tool compile -S main.go 
"".main STEXT size=163 args=0x0 locals=0x40
    ...
    0x0059 00089 (main.go:6)    MOVQ    AX, 16(SP)
    0x005e 00094 (main.go:6)    MOVQ    $1, 24(SP)
    0x0067 00103 (main.go:6)    MOVQ    $1, 32(SP)
    0x0070 00112 (main.go:6)    CALL    runtime.deferproc(SB)
    0x0075 00117 (main.go:6)    TESTL    AX, AX
    0x0077 00119 (main.go:6)    JNE    137
    0x0079 00121 (main.go:7)    XCHGL    AX, AX
    0x007a 00122 (main.go:7)    CALL    runtime.deferreturn(SB)
    0x007f 00127 (main.go:7)    MOVQ    56(SP), BP
    0x0084 00132 (main.go:7)    ADDQ    $64, SP
    0x0088 00136 (main.go:7)    RET
    0x0089 00137 (main.go:6)    XCHGL    AX, AX
    0x008a 00138 (main.go:6)    CALL    runtime.deferreturn(SB)
    0x008f 00143 (main.go:6)    MOVQ    56(SP), BP
    0x0094 00148 (main.go:6)    ADDQ    $64, SP
    0x0098 00152 (main.go:6)    RET
    ...
```

我們在前文提到 `defer` 關鍵字其實涉及了一系列的連鎖呼叫，內部 `runtime` 函式的呼叫就至少多了三步，分別是 `runtime.deferproc` 一次和 `runtime.deferreturn` 兩次。

而這還只是在執行時的顯式動作，另外編譯器做的事也不少，例如：

* 在 `deferproc` 階段（註冊延遲呼叫），還得取得/傳入目標函式地址、函式引數等等。
* 在 `deferreturn` 階段，需要在函式呼叫結尾處插入該方法的呼叫，同時若有被 `defer` 的函式，還需要使用 `runtime·jmpdefer` 進行跳轉以便於後續呼叫。

這一些動作途中還要涉及最小單元 `_defer` 的取得/生成， `defer` 和 `recover` 連結串列的邏輯處理和消耗等動作。

## Q\&A

最後討論的時候有提到 **“問題指的是本來就是用來執行 close() 一些操作的，然後說盡量不能用，例子就把 defer db.close() 前面的 defer 刪去了”** 這個疑問。

這是一個比較類似 “教科書” 式的說法，在一些入門教程中會潛移默化的告訴你在資源控制後加個 `defer` 延遲關閉一下。例如：

```go
resp, err := http.Get(...)
if err != nil {
    return err
}
defer resp.Body.Close()
```
但是一定得這麼寫嗎？其實並不，很多人給出的理由都是 “怕你忘記” 這種說辭，這沒有毛病。但需要認清場景，假設我的應用場景如下：

```go
resp, err := http.Get(...)
if err != nil {
    return err
}
defer resp.Body.Close()
// do something
time.Sleep(time.Second * 60)
```
嗯，一個請求當然沒問題，流量、併發一下子大了呢，那可能就是個災難了。你想想為什麼？從常見的 `defer` + `close` 的使用組合來講，用之前建議先看清楚應用場景，在保證無異常的情況下確保儘早關閉才是首選。如果只是小範圍呼叫很快就返回的話，偷個懶直接一套組合拳出去也未嘗不可。

## 結論

一個 `defer` 關鍵字實際上包含了不少的動作和處理，和你單純呼叫一個函式一條指令是沒法比的。而與對照物相比，它確確實實是有效能損耗，目前延遲呼叫的全部開銷大約在 50ns，但 `defer` 所提供的作用遠遠大於此，你從全域性來看，它的損耗非常小，並且官方還不斷地在最佳化中。

因此，對於 “Go defer 會有效能損耗，儘量不能用？” 這個問題，我認為**該用就用，應該及時關閉就不要延遲，在 hot paths 用時一定要想清楚場景**。

## 補充

最後補充上柴大的回覆：**“不是效能問題，defer 最大的功能是 Panic 後依然有效。如果沒有 defer，Panic 後就會導致 unlock 丟失，從而導致死鎖了”**，非常經典。
