# 8.1 fmt

## 前言

```go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("Hello World!")
}
```
標準開場見多了，那內部標準庫又是怎麼輸出這段英文的呢？今天一起來圍觀下原始碼吧 🤭

## 原型

```go
func Print(a ...interface{}) (n int, err error) {
    return Fprint(os.Stdout, a...)
}

func Println(a ...interface{}) (n int, err error) {
    return Fprintln(os.Stdout, a...)
}

func Printf(format string, a ...interface{}) (n int, err error) {
    return Fprintf(os.Stdout, format, a...)
}
```
* Print：使用預設格式說明符列印格式並寫入標準輸出。當兩者都不是字串時，在運算元之間新增空格
* Println：同上，不同的地方是始終在運算元之間新增空格，並附加換行符
* Printf：根據格式說明符進行格式化並寫入標準輸出

以上三類就是最常見的格式化 I/O 的方法，我們將基於此去進行拆解描述

## 執行流程

### 案例一：Print

在這裡我們使用 `Print` 方法做一個分析，便於後面的加深理解 😄

```go
func Print(a ...interface{}) (n int, err error) {
    return Fprint(os.Stdout, a...)
}
```
`Print` 使用預設格式說明符列印格式並寫入標準輸出。另外當兩者都為非空字串時將插入一個空格

#### 原型

```go
func Fprint(w io.Writer, a ...interface{}) (n int, err error) {
    p := newPrinter()
    p.doPrint(a)
    n, err = w.Write(p.buf)
    p.free()
    return
}
```
該函式一共有兩個形參：

* w：輸出流，只要實作 io.Writer 就可以（抽象）為流的寫入
* a：任意型別的多個值

#### 分析主幹流程

1、 p := newPrinter(): 申請一個臨時物件池（sync.Pool）

```go
var ppFree = sync.Pool{
    New: func() interface{} { return new(pp) },
}

func newPrinter() *pp {
    p := ppFree.Get().(*pp)
    p.panicking = false
    p.erroring = false
    p.fmt.init(&p.buf)
    return p
}
```
* ppFree.Get()：基於 sync.Pool 實作 \*pp 的臨時物件池，每次取得一定會返回一個新的 pp 物件用於接下來的處理
* \*pp.panicking：用於解決無限遞迴的 panic、recover 問題，會根據該引數在 catchPanic 及時掐斷
* \*pp.erroring：用於表示正在處理錯誤無效的 verb 識別符號，主要作用是防止呼叫 handleMethods 方法
* \*pp.fmt.init(\&p.buf)：初始化 fmt 設定，會設定 buf 並且清空 fmtFlags 標誌位

2、 p.doPrint(a): 執行約定的格式化動作（引數間增加一個空格、最後一個引數增加換行符）

```go
func (p *pp) doPrint(a []interface{}) {
    prevString := false
    for argNum, arg := range a {
        true && false
        isString := arg != nil && reflect.TypeOf(arg).Kind() == reflect.String
        // Add a space between two non-string arguments.
        if argNum > 0 && !isString && !prevString {
            p.buf.WriteByte(' ')
        }
        p.printArg(arg, 'v')
        prevString = isString
    }
}
```
可以看到底層透過判斷該入參，**同時**滿足以下條件就會新增分隔符（空格）：

* 當前入參為多個引數（例如：Slice）
* 當前入參不為 nil 且不為字串（透過反射確定）
* 當前入參不為首項或上一個入參不為字串

而在 `Print` 方法中，不需要指定格式符。實際上在該方法內直接指定為 `v`。也就是預設格式的值

```
p.printArg(arg, 'v')
```

1. w\.Write(p.buf): 寫入標準輸出（io.Writer）
2. \*pp.free(): 釋放已快取的內容。在使用完臨時物件後，會將 buf、arg、value 清空再重新存放到 ppFree 中。以便於後面再取出重用（利用 sync.Pool 的臨時物件特性）

### 案例二：Printf

#### 識別符號

**Verbs**

```
%v    the value in a default format
    when printing structs, the plus flag (%+v) adds field names
%#v    a Go-syntax representation of the value
%T    a Go-syntax representation of the type of the value
%%    a literal percent sign; consumes no value
%t    the word true or false
```

**Flags**

```
+    always print a sign for numeric values;
    guarantee ASCII-only output for %q (%+q)
-    pad with spaces on the right rather than the left (left-justify the field)
#    alternate format: add leading 0 for octal (%#o), 0x for hex (%#x);
    0X for hex (%#X); suppress 0x for %p (%#p);
    for %q, print a raw (backquoted) string if strconv.CanBackquote
    returns true;
    always print a decimal point for %e, %E, %f, %F, %g and %G;
    do not remove trailing zeros for %g and %G;
    write e.g. U+0078 'x' if the character is printable for %U (%#U).
' '    (space) leave a space for elided sign in numbers (% d);
    put spaces between bytes printing strings or slices in hex (% x, % X)
0    pad with leading zeros rather than spaces;
    for numbers, this moves the padding after the sign
```

詳細建議參見 [Godoc](https://golang.org/pkg/fmt/#hdr-Printing)

#### 原型

```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
    p := newPrinter()
    p.doPrintf(format, a)
    n, err = w.Write(p.buf)
    p.free()
    return
}
```
與 Print 相比，最大的不同就是 doPrintf 方法了。在這裡我們來詳細看看其程式碼，如下：

```go
func (p *pp) doPrintf(format string, a []interface{}) {
    end := len(format)
    argNum := 0         // we process one argument per non-trivial format
    afterIndex := false // previous item in format was an index like [3].
    p.reordered = false
formatLoop:
    for i := 0; i < end; {
        p.goodArgNum = true
        lasti := i
        for i < end && format[i] != '%' {
            i++
        }
        if i > lasti {
            p.buf.WriteString(format[lasti:i])
        }
        if i >= end {
            // done processing format string
            break
        }

        // Process one verb
        i++

        // Do we have flags?
        p.fmt.clearflags()
    simpleFormat:
        for ; i < end; i++ {
            c := format[i]
            switch c {
            case '#':   //'#'、'0'、'+'、'-'、' '
                ...
            default:
                if 'a' <= c && c <= 'z' && argNum < len(a) {
                    ...
                    p.printArg(a[argNum], rune(c))
                    argNum++
                    i++
                    continue formatLoop
                }

                break simpleFormat
            }
        }

        // Do we have an explicit argument index?
        argNum, i, afterIndex = p.argNumber(argNum, format, i, len(a))

        // Do we have width?
        if i < end && format[i] == '*' {
            ...
        }

        // Do we have precision?
        if i+1 < end && format[i] == '.' {
            ...
        }

        if !afterIndex {
            argNum, i, afterIndex = p.argNumber(argNum, format, i, len(a))
        }

        if i >= end {
            p.buf.WriteString(noVerbString)
            break
        }

        ...

        switch {
        case verb == '%': // Percent does not absorb operands and ignores f.wid and f.prec.
            p.buf.WriteByte('%')
        case !p.goodArgNum:
            p.badArgNum(verb)
        case argNum >= len(a): // No argument left over to print for the current verb.
            p.missingArg(verb)
        case verb == 'v':
            ...
            fallthrough
        default:
            p.printArg(a[argNum], verb)
            argNum++
        }
    }

    if !p.reordered && argNum < len(a) {
        ...
    }
}
```
#### 分析主幹流程

1. 寫入 % 之前的字元內容
2. 如果所有標誌位處理完畢（到達字元尾部），則跳出處理邏輯
3. （往後移）跳過 % ，開始處理其他 verb 標誌位
4. 清空（重新初始化） fmt 設定
5. 處理一些基礎的 verb 識別符號（simpleFormat）。如：'#'、'0'、'+'、'-'、' ' 以及**簡單的 verbs 識別符號（不包含精度、寬度和引數索引）。需要注意的是，若當前字元為簡單 verb 識別符號。則直接進行處理。完成後會直接後移到下一個字元**。其餘標誌位則變更 fmt 設定項，便於後續處理
6. 處理引數索引（argument index）
7. 處理引數寬度（width）
8. 處理引數精度（precision）
9. % 之後若不存在 verbs 識別符號則返回 `noVerbString`。值為 %!(NOVERB)
10. 處理特殊 verbs 識別符號（如：'%%'、'%#v'、'%+v'）、錯誤情況（如：引數索引指定錯誤、引數集個數與 verbs 識別符號數量不匹配）或進行格式化引數集
11. 常規流程處理完畢

在特殊情況下，若提供的引數集比 verb 識別符號多。fmt 將會貪婪檢查下去，將多出的引數集以特定的格式輸出，如下：

```go
fmt.Printf("%d", 1, 2, 3)
// 1%!(EXTRA int=2, int=3)
```

* 約定字首額外標誌：%!(EXTRA
* 當前引數的型別
* 約定格式符：=
* 當前引數的值（預設以 %v 格式化）
* 約定格式符：)

值得注意的是，當指定了引數索引或實際處理的引數小於入參的引數集時，就不會進行貪婪匹配來展示

### 案例三：Println

#### 原型

```go
func Fprintln(w io.Writer, a ...interface{}) (n int, err error) {
    p := newPrinter()
    p.doPrintln(a)
    n, err = w.Write(p.buf)
    p.free()
    return
}
```
在這個方法中，最大的區別就是 doPrintln，我們一起來看看，如下：

```go
func (p *pp) doPrintln(a []interface{}) {
    for argNum, arg := range a {
        if argNum > 0 {
            p.buf.WriteByte(' ')
        }
        p.printArg(arg, 'v')
    }
    p.buf.WriteByte('\n')
}
```
#### 分析主幹流程

* 迴圈入參的引數集，並以空格分隔
* 格式化當前引數，預設以 `%v` 對引數進行格式化
* 在結尾新增 `\n` 字元

## 如何格式化引數

在上例的執行流程分析中，可以看到格式化引數這一步是在 `p.printArg(arg, verb)` 執行的，我們一起來看看它都做了些什麼？

```go
func (p *pp) printArg(arg interface{}, verb rune) {
    p.arg = arg
    p.value = reflect.Value{}

    if arg == nil {
        switch verb {
        case 'T', 'v':
            p.fmt.padString(nilAngleString)
        default:
            p.badVerb(verb)
        }
        return
    }

    switch verb {
    case 'T':
        p.fmt.fmt_s(reflect.TypeOf(arg).String())
        return
    case 'p':
        p.fmtPointer(reflect.ValueOf(arg), 'p')
        return
    }

    // Some types can be done without reflection.
    switch f := arg.(type) {
    case bool:
        p.fmtBool(f, verb)
    case float32:
        p.fmtFloat(float64(f), 32, verb)
    ...
    case reflect.Value:
        if f.IsValid() && f.CanInterface() {
            p.arg = f.Interface()
            if p.handleMethods(verb) {
                return
            }
        }
        p.printValue(f, verb, 0)
    default:
        if !p.handleMethods(verb) {
            p.printValue(reflect.ValueOf(f), verb, 0)
        }
    }
}
```
在小節程式碼中可以看見，fmt 本身對不同的型別做了不同的處理。這樣子就避免了透過反射確定。相對的提高了效能

其中有兩個特殊的方法，分別是 `handleMethods` 和 `badVerb`，接下來分別來看看他們的作用是什麼

1、badVerb

它主要用於格式化並處理錯誤的行為。我們可以一起來看看，程式碼如下：

```go
func (p *pp) badVerb(verb rune) {
    p.erroring = true
    p.buf.WriteString(percentBangString)
    p.buf.WriteRune(verb)
    p.buf.WriteByte('(')
    switch {
    case p.arg != nil:
        p.buf.WriteString(reflect.TypeOf(p.arg).String())
        p.buf.WriteByte('=')
        p.printArg(p.arg, 'v')
    ...
    default:
        p.buf.WriteString(nilAngleString)
    }
    p.buf.WriteByte(')')
    p.erroring = false
}
```

在處理錯誤格式化時，我們可以對比以下例子：

```go
fmt.Printf("%s", []int64{1, 2, 3})
// [%!s(int64=1) %!s(int64=2) %!s(int64=3)]%
```

在 badVerb 中可以看到錯誤字串的處理主要分為以下部分：

* 約定字首錯誤標誌：%!
* 當前的格式化運算子
* 約定格式符：(
* 當前引數的型別
* 約定格式符：=
* 當前引數的值（預設以 %v 格式化）
* 約定格式符：)

2、handleMethods

```go
func (p *pp) handleMethods(verb rune) (handled bool) {
    if p.erroring {
        return
    }
    // Is it a Formatter?
    if formatter, ok := p.arg.(Formatter); ok {
        handled = true
        defer p.catchPanic(p.arg, verb)
        formatter.Format(p, verb)
        return
    }

    // If we're doing Go syntax and the argument knows how to supply it, take care of it now.
    ...

    return false
}
```
這個方法比較特殊，一般在自定義結構體和未知情況下進行呼叫。主要流程是：

* 若當前引數為錯誤 verb 識別符號，則直接返回
* 判斷是否實作了 Formatter&#x20;
* 實作，則利用自定義 Formatter 格式化引數
* 未實作，則最大程度的利用 Go syntax 預設規則去格式化引數

## 拓展

在 fmt 標準庫中可以透過自定義結構體來實作方法的自定義，大致如下幾種

### fmt.State

```go
type State interface {
    Write(b []byte) (n int, err error)

    Width() (wid int, ok bool)

    Precision() (prec int, ok bool)

    Flag(c int) bool
}
```
State 用於取得標誌位的狀態值，涉及如下：

* Write：將格式化完畢的字元寫入緩衝區中，等待下一步處理
* Width：返回寬度資訊和是否被設定
* Precision：返回精度資訊和是否被設定
* Flag：返回特殊標誌符（'#'、'0'、'+'、'-'、' '）是否被設定

### fmt.Formatter

```go
type Formatter interface {
    Format(f State, c rune)
}
```
Formatter 用於實作**自定義格式化方法**。可透過在自定義結構體中實作 Format 方法來實作這個目的

另外，可以透過 f 取得到當前識別符號的寬度、精度等狀態值。c 為 verb 識別符號，可以得到其動作是什麼

### fmt.Stringer

```go
type Stringer interface {
    String() string
}
```
當該物件為 String、Array、Slice 等型別時，將會呼叫 `String()` 方法對類字串進行格式化

### fmt.GoStringer

```go
type GoStringer interface {
    GoString() string
}
```
當格式化特定 verb 識別符號（%v）時，將呼叫 `GoString()` 方法對其進行格式化

## 總結

透過本文對 fmt 標準庫的分析，可以發現它有以下特點：

* 在拓展性方面，可以自定義格式化方法等
* 在完整度方面，儘可能的貪婪匹配，輸出引數集
* 在效能方面，每種不同的引數型別，都實作了不同的格式化處理操作
* 在效能方面，儘可能的最短匹配，格式化引數集

總的來說，fmt 標準庫有許多值得推敲的細節，希望你能夠在本文學到 😄
