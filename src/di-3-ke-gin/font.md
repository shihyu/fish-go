# 3.16 在圖片上繪製文字

專案地址：<https://github.com/EDDYCJY/go-gin-example>

## 知識點

* 字型庫使用
* 圖片合成

## 本文目標

主要實作**合併後的海報上繪製文字**的功能（這個需求也是常見的很了），內容比較簡單。

## 實作

這裡使用的是 [微軟雅黑](https://github.com/EDDYCJY/go-gin-example/blob/master/runtime/fonts/msyhbd.ttc) 的字型，請點選進行下載並**存放到 runtime/fonts 目錄**下（字型檔案佔 16 MB 大小）

### 安裝

```
$ go get -u github.com/golang/freetype
```

### 繪製文字

開啟 service/article\_service/article\_poster.go 檔案，增加繪製文字的業務邏輯，如下：

```go
type DrawText struct {
    JPG    draw.Image
    Merged *os.File

    Title string
    X0    int
    Y0    int
    Size0 float64

    SubTitle string
    X1       int
    Y1       int
    Size1    float64
}

func (a *ArticlePosterBg) DrawPoster(d *DrawText, fontName string) error {
    fontSource := setting.AppSetting.RuntimeRootPath + setting.AppSetting.FontSavePath + fontName
    fontSourceBytes, err := ioutil.ReadFile(fontSource)
    if err != nil {
        return err
    }

    trueTypeFont, err := freetype.ParseFont(fontSourceBytes)
    if err != nil {
        return err
    }

    fc := freetype.NewContext()
    fc.SetDPI(72)
    fc.SetFont(trueTypeFont)
    fc.SetFontSize(d.Size0)
    fc.SetClip(d.JPG.Bounds())
    fc.SetDst(d.JPG)
    fc.SetSrc(image.Black)

    pt := freetype.Pt(d.X0, d.Y0)
    _, err = fc.DrawString(d.Title, pt)
    if err != nil {
        return err
    }

    fc.SetFontSize(d.Size1)
    _, err = fc.DrawString(d.SubTitle, freetype.Pt(d.X1, d.Y1))
    if err != nil {
        return err
    }

    err = jpeg.Encode(d.Merged, d.JPG, nil)
    if err != nil {
        return err
    }

    return nil
}
```
這裡主要使用了 freetype 包，分別涉及如下細項：

1、freetype.NewContext：建立一個新的 Context，會對其設定一些預設值

```go
func NewContext() *Context {
    return &Context{
        r:        raster.NewRasterizer(0, 0),
        fontSize: 12,
        dpi:      72,
        scale:    12 << 6,
    }
}
```
2、fc.SetDPI：設定螢幕每英寸的解析度

3、fc.SetFont：設定用於繪製文字的字型

4、fc.SetFontSize：以磅為單位設定字型大小

5、fc.SetClip：設定剪裁矩形以進行繪製

6、fc.SetDst：設定目標影象

7、fc.SetSrc：設定繪製操作的源影象，通常為 [image.Uniform](https://golang.org/pkg/image/#Uniform)

```
var (
        // Black is an opaque black uniform image.
        Black = NewUniform(color.Black)
        // White is an opaque white uniform image.
        White = NewUniform(color.White)
        // Transparent is a fully transparent uniform image.
        Transparent = NewUniform(color.Transparent)
        // Opaque is a fully opaque uniform image.
        Opaque = NewUniform(color.Opaque)
)
```

8、fc.DrawString：根據 Pt 的座標值繪製給定的文字內容

### 業務邏輯

開啟 service/article\_service/article\_poster.go 方法，在 Generate 方法增加繪製文字的程式碼邏輯，如下：

```go
func (a *ArticlePosterBg) Generate() (string, string, error) {
    fullPath := qrcode.GetQrCodeFullPath()
    fileName, path, err := a.Qr.Encode(fullPath)
    if err != nil {
        return "", "", err
    }

    if !a.CheckMergedImage(path) {
        ...

        draw.Draw(jpg, jpg.Bounds(), bgImage, bgImage.Bounds().Min, draw.Over)
        draw.Draw(jpg, jpg.Bounds(), qrImage, qrImage.Bounds().Min.Sub(image.Pt(a.Pt.X, a.Pt.Y)), draw.Over)

        err = a.DrawPoster(&DrawText{
            JPG:    jpg,
            Merged: mergedF,

            Title: "Golang Gin 系列文章",
            X0:    80,
            Y0:    160,
            Size0: 42,

            SubTitle: "---煎鱼",
            X1:       320,
            Y1:       220,
            Size1:    36,
        }, "msyhbd.ttc")

        if err != nil {
            return "", "", err
        }
    }

    return fileName, path, nil
}
```
## 驗證

訪問生成文章海報的介面 `$HOST/api/v1/articles/poster/generate?token=$token`，檢查其生成結果，如下圖

![image](../images/af06b347ae49-qaaG0LE.jpg)

## 總結

在本章節在 [連載十五](https://github.com/EDDYCJY/blog/blob/master/golang/gin/2018-07-04-Gin%E5%AE%9E%E8%B7%B5-%E8%BF%9E%E8%BD%BD%E5%8D%81%E4%BA%94-%E7%94%9F%E6%88%90%E4%BA%8C%E7%BB%B4%E7%A0%81-%E5%90%88%E5%B9%B6%E6%B5%B7%E6%8A%A5.md) 的基礎上增加了繪製文字，在實作上並不困難，而這兩塊需求一般會同時出現，大家可以多加練習，瞭解裡面的邏輯和其他 API 😁

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
