# line_alexa_webhook_skill

Alexa skill to check and read messages such like LINE with webhook and DB

LINEの新着メッセージをWebhookを利用して確認します。
実は、Webhookで取得できる情報なら、LINEに限定されません。

## メッセージ取得の工程

具体的には以下の工程で処理が行われます。
このスキルとは別でユーザーが準備するべき項目も多いです。

1. LINEのメッセージをLINE BotのMessaging APIのWebhook機能でGASやphpを併用したDBなどに保存します。
2. あらかじめ、本スキルにDBからのメッセージ取得用のURLなどの情報を登録しておきます。具体的な方法は後述します。
3. 本スキルは呼び出された際に、登録されたURLにGETでアクセスして、保存されているメッセージを取得します。
    1件以上のメッセージを取得した場合、「LINEにn件のメッセージがあります」と発声します。
    「読んで」と応じることで、本スキルは取得したメッセージを読み上げます。
    「1件目、18時12分。今、駅に着いた。...2件目、(以下略)」の形式です。
    その後、登録されたURLに読み上げを完了した旨をGETで送信します。

## 情報の登録

本スキルがDBにアクセスするためには、いくつかの情報が必要です。
それらを登録するため、本スキルではアレクサの「やること」リストを経由します。

そのため、情報を登録する段階ではリストの読み込み・書き込みの権限を許可してください。

「ラインを確認を起動して登録して」と呼びかけると、本スキルの情報登録機能が起動します。
あるいは、情報が登録されていない状態で「ラインを確認」と呼びかけるだけでも、起動します。

情報登録機能が起動すると、まずやることリストに下記の必須項目をアイテムとして追加します。
やることリストは「アレクサアプリ下部のその他→リストとメモ→やること」でアクセスできます。
すべてのアイテムは「LINE_Check_」から始まり、はじめのコロン以降は記入例であり、
ユーザーによって変更されることが想定されています。

### 必須項目

- LINE_Check_URL_Request: https://hogehoge.com
    - WebhookのURL
- LINE_Check_Header_Authorization: Bearer Token
    - GET通信の際のHeadersのAuthorization項目の内容
    - Authorization項目が不要な場合は、適当な値(例えば「Bearer Token」のまま放置)でよい

### 任意項目 (必要に応じてユーザーが追加すれば、本スキルによって認識される)

- LINE_Check_Header_%KEY%: %VAL%
    - 「%KEY%」および「%VAL%」の部分は任意の文字列。(ただし「%KEY%」はコロンを含まない前提)
    - GET通信の際にHeadersの「%KEY%」項目の内容として「%VAL%」が追加される
    - 任意の個数追加できる

## GET通信の仕様

### メッセージを取得する段階

登録されたURLにGET通信を行います。
queryパラメータとして「command=obtainMessages」が追加されます。
返り値はjson形式で、下記の構造が想定されています。

```
{
    "messages":[
        {
            "content":%MESSAGE_TEXT%,
            "timestamp":%TIMESTAMP%
        }, {
            "content":%MESSAGE_TEXT2%,
            "timestamp":%TIMESTAMP2%
        }, ...
    ],
    "hash": %HASH_VALUE%
}
```

%MESSAGE_TEXT%の部分に読み上げるべきメッセージのテキストが、
%TIMESTAMP%の部分に各メッセージのタイムスタンプ(unix time)が入ることが想定されています。

%HASH_VALUE%の部分は省略しても構いません。
読み上げに成功した場合、この値が"hash"の値としてqueryパラメータに追加されます。

### メッセージの読み上げに成功した段階

登録されたURLにGET通信を行います。
queryパラメータとして「command=succeed」が追加されます。
メッセージ取得の段階でhash値が与えられていた場合、「hash=%HASH_VALUE%」も追加されます。

返り値の内容はスキルの応答に影響を与えません。

## TIPS

### DBでの処理
DBからメッセージを返す段階までに、メッセージの選別および整備が完了しているべきです。
(そのままではアレクサが正しく読み上げられない固有名詞の書き換えや、直近6時間のメッセージのみ選び出すなど)

特に、本スキルから読み上げに成功したGET通信を受信した時点で、該当メッセージには読み上げ完了済みのフラグを立てておきましょう。
さもないと、本スキルを起動するたびに同じメッセージを読み上げ続けることになります。

### 定期実行
本スキルをスキル内の機能だけで定期的に実行することは不可能でした。
定型アクションから、特定の時間ごとにアレクサに「ラインを確認を開いて自動で確認して」というフレーズを実行させてください。
「自動で」というワードを入れることで、取得したメッセージが0件の場合に無言でスキルを終了します。

### 起動フレーズ

「ラインを確認して」と呼びかけた際に「ラインを確認」を起動するようにしておいた方が使いやすいです。
