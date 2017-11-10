# 🔰Amazon Simple Queue Serviceを触ってみる　&　PowershellからSQSのAPIを叩いてみる

今回はAWS最古のサービスと言われるAmazonSQSを触ってみる。

- お品書き
  - Amazon SQSとはなんぞや
  - AWSコンソールからキューの作成をメッセージの送受信
  - PowershellからSQSのAPIを叩いてみる

## 🔰Amazon Simple Queue Serviceのドキュメント

公式のドキュメントは
[https://aws.amazon.com/jp/documentation/sqs/](https://aws.amazon.com/jp/documentation/sqs/)
にあるのでここをみればOK

## 🔰Amazon Simple Queue Serviceとはなんぞや

> Amazon Simple Queue Service（Amazon SQS）は、システムの他のコンポーネントとの間でメッセージまたはワークフローを処理するメッセージングキューサービスです。（公式ドキュメントより）

キューに対してメッセージを配信し、キューに配信されたメッセージを色々と利用することにより。
システム間の処理を分離することが肝っぽい。

>Amazon SQS は分散システムであるため、キューにあるメッセージが非常に少ない場合は、受信
したリクエストに対して空のレスポンスを表示する場合があります。この場合、リクエストを再
実行してメッセージを取得できます。アプリケーションの必要に応じて、メッセージを受信する
ためにショートポーリングまたはロングポーリング (p. 60)を使用する必要がある場合がありま
す。（公式ドキュメントより）

## 🔰キューの種類

- スタンダード キュー
- FIFO（先入れ先出し） キュー

※2017年6月28日現在、スタンダードキューは全てのリージョンでサービスしているが、FIFOキューは US East (N. Virginia), US East (Ohio), US West (Oregon), and EU (Ireland) リージョンのみ。

## 🔰標準キューとFIFOキュー

| 標準キュー        | FIFOキュー                   |
|--------------|---------------------------|
| 提供リージョン すべて  | 提供リージョン　一部※2017年06月28日現在  |
| スループット制限　無制限 | スループット制限　1秒あたり300トランザクション |
| 少なくとも 1 回の配信 | 1 回だけの処理                  |
| 送信順序は保証されない  | 送信順序は保証                   |

## 🔰ショートポーリングとロングポーリング

メッセージの受信動作は2種類ある。

- ショートポーリング(標準)
- ロングポーリング

## 🔰ショートポーリング(標準)

Amazon SQS は分散システムで複数のサーバにメッセージが格納されている。  
ショートポーリングでは複数あるサーバから幾つかのサーバがサンプリングされてそれらのサーバからのみメッセージが返される。  
なのでメッセージの受信リクエストに対して、メッセージがすべて取得できる事を保証していない。  
実際にはキューにメッセージが配信されてる状況でも、受信リクエストに対してレスポンスが空だったり一部だったりを返すケースがある。

## 🔰ロングポーリング

Amazon SQS は分散システムで複数のサーバにメッセージが格納されている。  
ロングポーリングでは複数あるサーバすべてに対してクエリを発行し、メッセージの取得を行う。  

公式のドキュメントに多くの場合ロングポーリングが推奨されるとある通り、基本的にメッセージの受信にはロングポーリングを使っておけば問題ないと思う。

[全般的な推奨事項](http://docs.aws.amazon.com/ja_jp/AWSSimpleQueueService/latest/SQSDeveloperGuide/general-recommendations.html)

ショートポーリングはよっぽどの要件がない限り使用する機会はなさそう。また使用するのならばかなりきっちり設計しないとダメっぽい。

## 🔰Amazon SQSを利用するのに念頭に置くべきこと

> Q: Amazon SQS では、メッセージが配信されることは保証されますか?
>
> 標準キューでは 1 回以上の配信が行われます。これは、各メッセージは少なくとも 1 回配信されることを意味します。
>
> FIFO キューではちょうど 1 回の処理が行われます。これは、各メッセージは 1 回配信されて、ユーザーがメッセージを処理して削除するまで使用可能な状態に留まることを意味します。キューでメッセージの重複が起きることはありません。

[AWSSQSのよくある質問](https://aws.amazon.com/jp/sqs/faqs/)より。

上記の仕様なため、冪等性をよく考えて実装する必要がある。

またメッセージの受信についても、ショートポーリングだとメッセージの取得が保証されていないので使用する場合はそれを考慮した設計で実装してあげる必要がある。

まとめると

- 標準キューだと処理順や配信回数が保証されないためそれを考慮した実装を行うこと！
- メッセージの取得は基本ロングポーリングを使う！
- ショートポーリングは使いこなすのが難しそう！

色々と癖があるので最初に上記の事を念頭に置かないとまともな実装は出来ない……と思われる。

## 🔰AWSコンソールからキューを作ってみる(標準キュー)

公式ドキュメントの入門を見るとまずは、AWSコンソールからキューを作成しているので試してみる。

新しいキューの作成（標準キュー）  
![](./image/awsconsole.create.que.step001.png)

※FIFOキューの場合はキューの名前を.fifoで終わらせる必要があるらしい。
ちなみに、画像だとメッセージ受信時間が標準の0のままになっている。

この値はキューのデフォルト設定で利用するWaitTimeSecondsを指定しており

WaitTimeSecondsの設定によりポーリングの設定が下記のようになる。

- 0の場合はショートポーリング
- 0以外の場合はロングポーリング

APIからWaitTimeSecondsを指定しない場合はこのキューのデフォルト設定が適用され、指定した場合は指定した値が優先される。
ここの値を0以外にしておけば、APIを叩く時にWaitTimeSecondsを明示的に0を指定した場合のみショートポーリングとなります。  
![](./image/awsconsole.create.que.step002.png)

作成されたキュー  
![](./image/awsconsole.create.que.step003.png)

## 🔰AWSコンソールからキューに対するアクセス権設定

AWS SQSではキューに対してアクセス権を設定することが出来る。
ここでは全てのユーザに既存キューのURL取得を許可する。

アクセスの許可  
![](./image/awsconsole.auth.que.step001.png)

TestQueへのアクセス許可の追加  
![](./image/awsconsole.auth.que.step002.png)

作成されたTestQue  
![](./image/awsconsole.auth.que.step003.png)

## 🔰AWSコンソールから作成したキューにメッセージを送信

作成したキューに対してメッセージの送信

メッセージの送信  
![](./image/awsconsole.send.message.step001.png)

TestQueへのメッセージの送信  
![](./image/awsconsole.send.message.step002.png)

TestQueへのメッセージの送信  
![](./image/awsconsole.send.message.step003.png)

利用可能なメッセージが1
![](./image/awsconsole.send.message.step004.png)

## 🔰AWSコンソールから作成したキューのメッセージを表示

キューに送信されたメッセージの取得

メッセージの表示／削除  
![](./image/awsconsole.read.message.step001.png)

初回時のメッセージ  
![](./image/awsconsole.read.message.step002.png)

キューのメッセージを表示  
![](./image/awsconsole.read.message.step003.png)

## 🔰AWSコンソールから作成したキューのメッセージを削除

キューに送信されたメッセージの削除

削除対象となるメッセージにチェックを付けて削除  
![](./image/awsconsole.delete.message.step001.png)

メッセージの削除  
![](./image/awsconsole.delete.message.step002.png)

削除したメッセージが表示から削除された  
![](./image/awsconsole.delete.message.step003.png)

利用可能なメッセージが0  
![](./image/awsconsole.delete.message.step004.png)

## 🔰AWSコンソールから作成したキューの削除

キュー自体の削除。

キューを利用しておらず近い将来も使う予定がない場合はキューを削除するのがAWS推奨。

キューの削除  
![](./image/awsconsole.delete.que.step001.png)

キューの削除  
![](./image/awsconsole.delete.que.step002.png)

キューが削除された事が確認できる
![](./image/awsconsole.delete.que.step003.png)

## 🔰PowerShellからSQSのAPIを叩いてみる

[Amazon SQS API を使用する](http://docs.aws.amazon.com/ja_jp/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-working-with-apis.html)

### 🔰APIのエンドポイント

東京リージョンの場合

- 'sqs.ap-northeast-1.amazonaws.com'

### 🔰GETリクエストでメッセージの送信を行う場合

まずは単純な所から。
全てのユーザに全てのアクションを許可したキューを作成してAPIでメッセージの送信。
認証やら署名をやるのはとりえあずつながってから！

APITestQueを作成  
![](./image/send.create.apitestque.png)

アクセス許可の追加  
![](./image/send.auth.apitestque.step001.png)

とりえず全開放。  
![](./image/send.auth.apitestque.step002.png)

キューの詳細にURLが出ているので控える  
![](./image/send.uri.apitestque.png)

```powershell
$uri = "https://sqs.ap-northeast-1.amazonaws.com/************/APITestQue?Action=SendMessage&MessageBody=HelloWorld&Version=2012-11-05"

#invoke-restmethodでAPIを叩く
$result = Invoke-RestMethod -Uri $uri -Method Get

```

実行  
![](./image/send.irm.getmethod.step001.png)

登録されたメッセージの確認  
![](./image/send.irm.getmethod.step002.png)

メッセージの表示  
![](./image/send.irm.getmethod.step003.png)

ちなみに正常な場合とエラーの場合のレスポンスについては下記に説明がありました。
[http://docs.aws.amazon.com/ja_jp/AWSSimpleQueueService/latest/SQSDeveloperGuide/UnderstandingResponses.html](http://docs.aws.amazon.com/ja_jp/AWSSimpleQueueService/latest/SQSDeveloperGuide/UnderstandingResponses.html)

### 🔰POSTリクエストでメッセージの送信を行う場合

Amazon SQSではPOSTリクエストも受け付けている。

```powershell
#APIで利用するBODYを設定する
$body = @{}

#ActionでSendMessageを指定
$body.add("Action","SendMessage")
#メッセージ本文をセット
$body.add("MessageBody","HelloWorld(POST)")
#APIバージョンは2012-11-05
$body.add("Version","2012-11-05")

#控えておいたurlをセット
$uri = "https://sqs.ap-northeast-1.amazonaws.com/************/APITestQue"

#invoke-restmethodでAPIを叩く
$result = Invoke-RestMethod -Uri $uri -Body $body -Method Post -ContentType "application/x-www-form-urlencoded"

```

実行  
![](./image/send.irm.postmethod.step001.png)

登録されたメッセージの確認  
![](./image/send.irm.postmethod.step002.png)

### 🔰POSTリクエストでメッセージ送信して送信したメッセージの受信を行う場合(ロングポーリング)

```powershell
#受信する用のメッセージを送信

#uri設定
$uri = "https://sqs.ap-northeast-1.amazonaws.com/************/APITestQue"

#メッセージ配信用のBODYを設定する
$body = @{}

#ActionでSendMessageを指定
$body.add("Action","SendMessage")
#メッセージ本文をセット
$body.add("MessageBody","TestMessage")
#APIバージョンは2012-11-05
$body.add("Version","2012-11-05")

#invoke-restmethodでAPIを叩く
$result = Invoke-RestMethod -Uri $uri -Body $body -Method Post -ContentType "application/x-www-form-urlencoded"

#登録されたMessageIdを控える
$messageID = $result.SendMessageResponse.SendMessageResult.MessageId

#BODYのクリア
$body.clear()
#メッセージの受信で利用するBODYを設定する
#ActionでReceiveMessageを指定
$body.add("Action","ReceiveMessage")
#WaitTimeSecondsパラメータを使うとロングポーリングが有効になる
$body.add("WaitTimeSeconds",20)
#VisibilityTimeoutでメッセージの可視性タイムアウトを設定（一度読まれたキューが、再度読み込まれない期間を設定）
$body.add("VisibilityTimeout",500)
#最大10件のメッセージ
$body.add("MaxNumberOfMessages","10")
#全ての情報を取得
$body.add("AttributeName","All")
#APIバージョンは2012-11-05
$body.add("Version","2012-11-05")

#送信したメッセージが取得出来るまでメッセージの取得を行う
#（取得したメッセージのメッセージIDを確認して、送信した物が拾えるまでAPI実行し続ける）

[boolean]$isCatchTheTarget = $False

#ターゲットが取得出来るまでループ
while( (! $isCatchTheTarget) ){
    #invoke-restmethodでAPIを叩く
    $result = Invoke-RestMethod -Uri $uri -Body $body -Method Post -ContentType "application/x-www-form-urlencoded"

    #対象が取得出来ているかチェック
    $isCatchTheTarget = $result.ReceiveMessageResponse.ReceiveMessageResult.Message.MessageId -contains $messageID
}

#送信したメッセージを抜き出し
$resultMessage = $result.ReceiveMessageResponse.ReceiveMessageResult.Message | ?{ $_.MessageId -eq $messageID }

```

### 🔰POSTリクエストでキューのメッセージを全件取得する場合(ロングポーリング)

```powershell
#受信する用のメッセージを送信

#uri設定
$uri = "https://sqs.ap-northeast-1.amazonaws.com/************/APITestQue"

#メッセージの受信で利用するBODYを設定する
$body = @{}
#ActionでReceiveMessageを指定
$body.add("Action","ReceiveMessage")
#WaitTimeSecondsパラメータを使うとロングポーリングが有効になる
$body.add("WaitTimeSeconds",5)
#VisibilityTimeoutでメッセージの可視性タイムアウトを設定（一度読まれたキューが、再度読み込まれない期間を設定）
$body.add("VisibilityTimeout",500)
#最大10件のメッセージ
$body.add("MaxNumberOfMessages","10")
#全ての情報を取得
$body.add("AttributeName","All")
#APIバージョンは2012-11-05
$body.add("Version","2012-11-05")

#メッセージが取得出来る限り取得する
#最後メッセージが取得出来なくなった時はWaitTimeSeconds待機して終了する
#キューが全件読み終わる前に、VisibilityTimeoutになると再度読み込まれてしまうので
#キューの件数とVisibilityTimeoutの設定によっては下手するとだいぶ読み込み続けるかも？
#普通はキューを読み込み処理が終わったら削除すると思うので……
[boolean]$isEmpty = $False

while(! $isEmpty ){
    #invoke-restmethodでAPIを叩く
    $result = Invoke-RestMethod -Uri $uri -Body $body -Method Post -ContentType "application/x-www-form-urlencoded"
    $isEmpty = [string]::IsNullOrEmpty($result.ReceiveMessageResponse.ReceiveMessageResult)
}

```

### 🔰POSTリクエストでメッセージの削除をする場合

```powershell
#受信する用のメッセージを送信

#uri設定
$uri = "https://sqs.ap-northeast-1.amazonaws.com/************/APITestQue"

#メッセージ配信用のBODYを設定する
$body = @{}

#ActionでSendMessageを指定
$body.add("Action","SendMessage")
#メッセージ本文をセット
$body.add("MessageBody","TestMessage")
#APIバージョンは2012-11-05
$body.add("Version","2012-11-05")

#invoke-restmethodでAPIを叩く
$result = Invoke-RestMethod -Uri $uri -Body $body -Method Post -ContentType "application/x-www-form-urlencoded"

#登録されたMessageIdを控える
$messageID = $result.SendMessageResponse.SendMessageResult.MessageId

#BODYのクリア
$body.clear()
#メッセージの受信で利用するBODYを設定する
#ActionでReceiveMessageを指定
$body.add("Action","ReceiveMessage")
#WaitTimeSecondsパラメータを使うとロングポーリングが有効になる
$body.add("WaitTimeSeconds",20)
#VisibilityTimeoutでメッセージの可視性タイムアウトを設定（一度読まれたキューが、再度読み込まれない期間を設定）
$body.add("VisibilityTimeout",500)
#最大10件のメッセージ
$body.add("MaxNumberOfMessages","10")
#全ての情報を取得
$body.add("AttributeName","All")
#APIバージョンは2012-11-05
$body.add("Version","2012-11-05")

#送信したメッセージが取得出来るまでメッセージの取得を行う
#（取得したメッセージのメッセージIDを確認して、送信した物が拾えるまでAPI実行し続ける）

[boolean]$isCatchTheTarget = $False

#ターゲットが取得出来るまでループ
while( (! $isCatchTheTarget) ){
    #invoke-restmethodでAPIを叩く
    $result = Invoke-RestMethod -Uri $uri -Body $body -Method Post -ContentType "application/x-www-form-urlencoded"

    #対象が取得出来ているかチェック
    $isCatchTheTarget = $result.ReceiveMessageResponse.ReceiveMessageResult.Message.MessageId -contains $messageID
}

#送信したメッセージを抜き出し
$resultMessage = $result.ReceiveMessageResponse.ReceiveMessageResult.Message | ?{ $_.MessageId -eq $messageID }

```

## 🔰総評

Amazon SQS はキューサービスなのでキューを取り扱える訳だが。

下名は勝手にFIFOだと勘違いしてたので、最初に触った時に動きに戸惑ったりした。

また分散システムとしてメッセージが保存されているので、メッセージの取得リクエストを送っても必ずしもメッセージの取得が保証されていない場合があったり。メッセージの取得が空振る場合があったり。可視性タイムアウトの概念だったり。

分散システム？　という感じだったので。そこら辺でも戸惑ってみたり。

公式ドキュメントをざっくりとでも読んで、ある程度触ってみて、実際の動きを見て。

それから実装をはじめるのが安全。

というか基本的にはロングポーリングを指定しないと結構大変な実装になりそうな気もする？

下名が勝手にこういう動作だろうと当たりを付けていた所が全部外れていた感じなので。
他の人はそんなに戸惑わない可能性もなきにしもあらず。
