# 2.2 如此，用dep取得私有庫

## 介紹

`dep`是一個依賴管理工具。它需要1.9或更新的`Golang`版本才能編譯

`dep`已經能夠在生產環節安全使用，但還在官方的試驗階段，也就是還不在`go tool`中。但我想是遲早的事 :=)

指南和參考資料，請參閱[文件](https://golang.github.io/dep/)

## 取得私有庫

我們常用的`git`方式有兩種，第一種是透過`ssh`，第二種是`https`

本文中我們以`gitlab.com`為案例，建立一個`private`的私有倉庫

### 透過`ssh`方式

首先我們需要在本機上生成`ssh-key`，若沒有生成過可右拐[傳送門](https://segmentfault.com/a/1190000013450267)

得到需要使用的`ssh-key`後，我們開啟我們的`gitlab.com`，複製貼上入我們的`Settings` -> `SSH Keys`中

![image](https://sfault-image.b0.upaiyun.com/105/328/1053288135-5a96d458e9145)

新增成功後，我們直接在`Gopkg.toml`裡設定好我們的引數

```
[[constraint]]
  branch = "master"
  name = "gitlab.com/eddycjy/test"
  source = "git@gitlab.com:EDDYCJY/test.git"
```

在拉取資源前，要注意假設你是第一次用該`ssh-key`拉取，需要先手動用`git clone`拉取一遍，同意`ECDSA key fingerprint`：

```
$ git clone git@gitlab.com:EDDYCJY/test.git
Cloning into 'test'...
The authenticity of host 'gitlab.com (52.167.219.168)' can't be established.
ECDSA key fingerprint is xxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no)? yes
...
```

接下來，我們在專案下**直接執行`dep ensure`就可以拉取**下來了！

#### 問題

1. 假設你是第一次，又沒有執行上面那一步（`ECDSA key fingerprint`），會一直卡住
2. 正確的反饋應當是執行完命令後沒有錯誤，但如果出現該錯誤提示，那說明該儲存倉庫沒有被納入`dep`中（例：`gitee`）

   \`\`\` $ dep ensure

The following issues were found in Gopkg.toml:

unable to deduce repository and source type for "xxxx": unable to read metadata: go-import metadata not found

ProjectRoot name validation failed

```
### 通过`https`方式

我们直接在`Gopkg.toml`里配置好我们的参数
```

\[\[constraint]] branch = "master" name = "gitlab.com/eddycjy/test" source = "[https://{username}:{password}@gitlab.com](https://%7Busername%7D:%7Bpassword%7D@gitlab.com)"

\`\`\`

主要是修改`source`的設定項，`username`填寫在`gitlab`的使用者名稱，`password`為密碼

最後回到專案下**執行`dep ensure`拉取資源**就可以了

## 最後

`dep`目前還是官方試驗階段，還可能存在變數，多加註意
