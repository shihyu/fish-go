# 1.12 Go1.13 defer 的效能是如何提高的？

最近 Go1.13 終於釋出了，其中一個值得關注的特性就是 **defer 在大部分的場景下效能提升了30%**，但是官方並沒有具體寫是怎麼提升的，這讓大家非常的疑惑。而我因為之前寫過[《深入理解 Go defer》](https://book.eddycjy.com/golang/defer/defer.html) 和 [《Go defer 會有效能損耗，儘量不要用？》](https://book.eddycjy.com/golang/talk/defer-loss.html) 這類文章，因此我挺感興趣它是做了什麼改變才能得到這樣子的結果，所以今天和大家一起探索其中奧妙。

## 一、測試

### Go1.12

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

### Go1.13

```
$ go test -bench=. -benchmem -run=none
goos: darwin
goarch: amd64
pkg: github.com/EDDYCJY/awesomeDefer
BenchmarkDoDefer-4          15986062            74.7 ns/op          48 B/op           1 allocs/op
BenchmarkDoNotDefer-4       29231842            40.3 ns/op          48 B/op           1 allocs/op
PASS
ok      github.com/EDDYCJY/awesomeDefer    3.444s
```

在開場，我先以不標準的測試基準驗證了先前的測試用例，確確實實在這兩個版本中，`defer` 的效能得到了提高，但是看上去似乎不是百分百提高 30 %。

## 二、看一下

### 之前（Go1.12）

```
    0x0070 00112 (main.go:6)    CALL    runtime.deferproc(SB)
    0x0075 00117 (main.go:6)    TESTL    AX, AX
    0x0077 00119 (main.go:6)    JNE    137
    0x0079 00121 (main.go:7)    XCHGL    AX, AX
    0x007a 00122 (main.go:7)    CALL    runtime.deferreturn(SB)
    0x007f 00127 (main.go:7)    MOVQ    56(SP), BP
```

### 現在（Go1.13）

```
    0x006e 00110 (main.go:4)    MOVQ    AX, (SP)
    0x0072 00114 (main.go:4)    CALL    runtime.deferprocStack(SB)
    0x0077 00119 (main.go:4)    TESTL    AX, AX
    0x0079 00121 (main.go:4)    JNE    139
    0x007b 00123 (main.go:7)    XCHGL    AX, AX
    0x007c 00124 (main.go:7)    CALL    runtime.deferreturn(SB)
    0x0081 00129 (main.go:7)    MOVQ    112(SP), BP
```

從彙編的角度來看，像是 `runtime.deferproc` 改成了 `runtime.deferprocStack` 呼叫，難道是做了什麼最佳化，我們**抱著疑問**繼續看下去。

## 三、觀察原始碼

### \_defer

```go
type _defer struct {
    siz     int32
    siz     int32 // includes both arguments and results
    started bool
    heap    bool
    sp      uintptr // sp at time of defer
    pc      uintptr
    fn      *funcval
    ...
```
相較於以前的版本，最小單元的 `_defer` 結構體主要是新增了 `heap` 欄位，用於標識這個 `_defer` 是在堆上，還是在棧上進行分配，其餘欄位並沒有明確變更，那我們可以把聚焦點放在 `defer` 的堆疊分配上了，看看是做了什麼事。

### deferprocStack

```go
func deferprocStack(d *_defer) {
    gp := getg()
    if gp.m.curg != gp {
        throw("defer on system stack")
    }

    d.started = false
    d.heap = false
    d.sp = getcallersp()
    d.pc = getcallerpc()

    *(*uintptr)(unsafe.Pointer(&d._panic)) = 0
    *(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))
    *(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

    return0()
}
```
這一塊程式碼挺常規的，主要是取得呼叫 `defer` 函式的函式棧指標、傳入函式的引數具體地址以及PC（程式計數器），這塊在前文 [《深入理解 Go defer》](https://book.eddycjy.com/golang/defer/defer.html) 有詳細介紹過，這裡就不再贅述了。

那這個 `deferprocStack` 特殊在哪呢，我們可以看到它把 `d.heap` 設定為了 `false`，也就是代表 `deferprocStack` 方法是針對將 `_defer` 分配在棧上的應用場景的。

### deferproc

那麼問題來了，它又在哪裡處理分配到堆上的應用場景呢？

```go
func newdefer(siz int32) *_defer {
    ...
    d.heap = true
    d.link = gp._defer
    gp._defer = d
    return d
}
```
那麼 `newdefer` 是在哪裡呼叫的呢，如下：

```go
func deferproc(siz int32, fn *funcval) { // arguments of fn follow fn
    ...
    sp := getcallersp()
    argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
    callerpc := getcallerpc()

    d := newdefer(siz)
    ...
}
```
非常明確，先前的版本中呼叫的 `deferproc` 方法，現在被用於對應分配到堆上的場景了。

### 小結

* 第一點：可以確定的是 `deferproc` 並沒有被去掉，而是流程被優化了。
* 第二點：編譯器會根據應用場景去選擇使用 `deferproc` 還是 `deferprocStack` 方法，他們分別是針對分配在堆上和棧上的使用場景。

## 四、編譯器如何選擇

### esc

```go
// src/cmd/compile/internal/gc/esc.go
case ODEFER:
    if e.loopdepth == 1 { // top level
        n.Esc = EscNever // force stack allocation of defer record (see ssa.go)
        break
    }
```

### ssa

```
// src/cmd/compile/internal/gc/ssa.go
case ODEFER:
    d := callDefer
    if n.Esc == EscNever {
        d = callDeferStack
    }
    s.call(n.Left, d)
```

### 小結

這塊結合來看，核心就是當 `e.loopdepth == 1` 時，會將逃逸分析結果 `n.Esc` 設定為 `EscNever`，也就是將 `_defer` 分配到棧上，那這個 `e.loopdepth` 到底又是何方神聖呢，我們再詳細看看程式碼，如下：

```go
// src/cmd/compile/internal/gc/esc.go
type NodeEscState struct {
    Curfn             *Node
    Flowsrc           []EscStep 
    Retval            Nodes    
    Loopdepth         int32  
    Level             Level
    Walkgen           uint32
    Maxextraloopdepth int32
}
```
這裡重點檢視 `Loopdepth` 欄位，目前它共有三個值標識，分別是:

* -1：全域性。
* 0：返回變數。
* 1：頂級函式，又或是內部函式的不斷增長值。

這個讀起來有點繞，結合我們上述 `e.loopdepth == 1` 的表述來看，也就是當 `defer func` 是頂級函式時，將會分配到棧上。但是若在 `defer func` 外層出現顯式的迭代迴圈，又或是出現隱式迭代，將會分配到堆上。其實深層表示的還是迭代深度的意思，我們可以來證實一下剛剛說的方向，顯式迭代的程式碼如下：

```go
func main() {
    for p := 0; p < 10; p++ {
        defer func() {
            for i := 0; i < 20; i++ {
                log.Println("EDDYCJY")
            }
        }()
    }
}
```
檢視彙編情況：

```
$ go tool compile -S main.go
"".main STEXT size=122 args=0x0 locals=0x20
    0x0000 00000 (main.go:15)    TEXT    "".main(SB), ABIInternal, $32-0
    ...
    0x0048 00072 (main.go:17)    CALL    runtime.deferproc(SB)
    0x004d 00077 (main.go:17)    TESTL    AX, AX
    0x004f 00079 (main.go:17)    JNE    83
    0x0051 00081 (main.go:17)    JMP    33
    0x0053 00083 (main.go:17)    XCHGL    AX, AX
    0x0054 00084 (main.go:17)    CALL    runtime.deferreturn(SB)
    ...
```

顯然，最終 `defer` 呼叫的是 `runtime.deferproc` 方法，也就是分配到堆上了，沒毛病。而隱式迭代的話，你可以藉助 `goto` 語句去實作這個功能，再自己驗證一遍，這裡就不再贅述了。

## 總結

從分析的結果上來看，官方說明的 Go1.13 defer 效能提高 30%，主要來源於其延遲物件的堆疊分配規則的改變，措施是由編譯器透過對 `defer` 的 `for-loop` 迭代深度進行分析，如果 `loopdepth` 為 1，則設定逃逸分析的結果，將分配到棧上，否則分配到堆上。

的確，我個人覺得對大部分的使用場景來講，是優化了不少，也解決了一些人吐槽 `defer` 效能 “差” 的問題。另外，我想從 Go1.13 起，你也需要稍微瞭解一下它這塊的機制，別隨隨便便就來個狂野版巢狀迭代 `defer`，可能沒法效能最大化。

如果你還想了解更多細節，可以看看 `defer` 這塊的的[提交內容](https://github.com/golang/go/commit/fff4f599fe1c21e411a99de5c9b3777d06ce0ce6)，官方的測試用例也包含在裡面。
