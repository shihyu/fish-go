# 4.9 gRPC Deadlines

## 前言

在前面的章節中，已經介紹了 gRPC 的基本用法。那你想想，讓它這麼裸跑真的沒問題嗎？

那麼，肯定是有問題了。今天將介紹 gRPC Deadlines 的用法，這一個必備技巧。內容也比較簡單

## Deadlines

Deadlines 意指截止時間，在 gRPC 中強調 TL;DR（Too long, Don't read）並建議**始終設定截止日期**，為什麼呢？

### 為什麼要設定

當未設定 Deadlines 時，將採用預設的 DEADLINE\_EXCEEDED（這個時間非常大）

如果產生了阻塞等待，就會造成大量正在進行的請求都會被保留，並且所有請求都有可能達到最大超時

這會使服務面臨資源耗盡的風險，例如記憶體，這會增加服務的延遲，或者在最壞的情況下可能導致整個程序崩潰

## gRPC

### Client

```go
func main() {
    ...
    ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(time.Duration(5 * time.Second)))
    defer cancel()

    client := pb.NewSearchServiceClient(conn)
    resp, err := client.Search(ctx, &pb.SearchRequest{
        Request: "gRPC",
    })
    if err != nil {
        statusErr, ok := status.FromError(err)
        if ok {
            if statusErr.Code() == codes.DeadlineExceeded {
                log.Fatalln("client.Search err: deadline")
            }
        }

        log.Fatalf("client.Search err: %v", err)
    }

    log.Printf("resp: %s", resp.GetResponse())
}
```
* context.WithDeadline：會返回最終上下文截止時間。第一個形參為父上下文，第二個形參為調整的截止時間。若父級時間早於子級時間，則以父級時間為準，否則以子級時間為最終截止時間

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // The current deadline is already sooner than the new one.
        return WithCancel(parent)
    }
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    propagateCancel(parent, c)
    dur := time.Until(d)
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded) // deadline has already passed
        return c, func() { c.cancel(true, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}
```
* context.WithTimeout：很常見的另外一個方法，是便捷操作。實際上是對於 WithDeadline 的封裝

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
```
* status.FromError：返回 GRPCStatus 的具體錯誤碼，若為非法，則直接返回 `codes.Unknown`

### Server

```go
type SearchService struct{}

func (s *SearchService) Search(ctx context.Context, r *pb.SearchRequest) (*pb.SearchResponse, error) {
    for i := 0; i < 5; i++  {
        if ctx.Err() == context.Canceled {
            return nil, status.Errorf(codes.Canceled, "SearchService.Search canceled")
        }

        time.Sleep(1 * time.Second)
    }

    return &pb.SearchResponse{Response: r.GetRequest() + " Server"}, nil
}

func main() {
    ...
}
```
而在 Server 端，由於 Client 已經設定了截止時間。Server 勢必要去檢測它

否則如果 Client 已經結束掉了，Server 還傻傻的在那執行，這對資源是一種極大的浪費

因此在這裡需要用 `ctx.Err() == context.Canceled` 進行判斷，為了模擬場景我們加了迴圈和睡眠 🤔

### 驗證

重新啟動 server.go 和 client.go，得到結果：

```
$ go run client.go
2018/10/06 17:45:55 client.Search err: deadline
exit status 1
```

## 總結

本章節比較簡單，你需要知道以下知識點：

* 怎麼設定 Deadlines
* 為什麼要設定 Deadlines

你要清楚地明白到，gRPC Deadlines 是很重要的，否則這小小的功能點就會要了你生產的命 🤫

## 參考

### 本系列示例程式碼

* [go-grpc-example](https://github.com/EDDYCJY/go-grpc-example)

### 資料

* \[gRPC and Deadlines

  ]\(<https://grpc.io/blog/deadlines>)
