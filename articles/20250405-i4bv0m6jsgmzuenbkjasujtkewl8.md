---
title: "メール一斉送信バッチ処理を考える"
emoji: "✉️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

[【反省】バッチの実装を誤って、盛大にご迷惑をおかけしてしまった話](https://zenn.dev/maya_honey/articles/efbcb8f6bd01f0)
を読んで、メール送信バッチ処理難しいよなぁと改めて思ったので、どんな実装が考えられるかまとめてみようと思いました。  
RDBでのトランザクション管理と、エラーハンドリングを考えます。
網羅したいというモチベーションはなくて、自分の道具箱の整理くらいの話です。  
JavaScriptっぽい疑似言語で書いてみます。

## 1件ずつメール送信する

ある状態のユーザのメールアドレスを抽出し、別の状態を更新したあとでメールを送信する、という処理を考えます。  
SMTPでメールサーバに送信リクエストをするsend_mail関数を都度呼び出す単純な処理です。
状態の更新処理が失敗した場合はメール送信もスキップし、メール送信処理が失敗したらrollbackします。  
これにより、同じユーザに複数回メールが送信されることを防ぎます。  
一方で、他のユーザへはメール送信処理を継続し、最後にエラーを通知して、別途対応とすることにします。

```javascript
// 実装は省略
function get_mails() {
}

function send_mail(mail) {
}

function begin() {
}

function update(mail) {
}

function commit() {
}

function rollback() {
}

function notify(errors) {
}

const mails = get_mails()

errors = []
for (let mail of mails) {
    try {
        begin()
        update(mail)
        send_mail(mail)
        commit()
    } catch (e) {
        errors.push(mail)
        rollback()
    }
}

notify(errors)
```

### pros

- メール送信が同期的に行われ、バッチ処理のエラー・メール送信のエラーを同様に扱える

### cons

- 直列で実行されるため、送信件数が増えると処理時間が伸びる

## 複数のメールを一括で送信処理に渡す

送信件数が増えた場合に備え、処理の高速化を考えます。  
例えば、SendGridは1000件まで1回のリクエストで送信できるようです。  
[SendGridを使って短時間に大量のメールを送るための方法 | SendGridブログ](https://sendgrid.kke.co.jp/blog/?p=1300)  
1000件ごとにひとかたまり(=チャンク)として、メール送信APIを呼び出してみます。

```JavaScript
function get_mails() {
}

function send_mail_by_api(mails) {
}

function begin() {
}

function update(mails) {
}

function commit() {
}

function rollback() {
}

function notify(errors) {
}

const mails = get_mails()

const chunkSize = 1000;
const errors = [];
for (let i = 0; i < mails.length; i += chunkSize) {
    const chunk = mails.slice(i, i + chunkSize);
    try {
        begin()
        update(chunk)
        send_mail_by_api(chunk)
        commit()
    } catch (e) {
        rollback()
        errors.push(...chunk)
    }
}

notify(errors)
```

### pros

- チャンクごとに処理されるため、送信件数が増えても処理時間が伸びづらい

### cons

- エラー時はチャンクの単位で再処理する必要がある
- send_mail_by_api関数でのAPI呼び出しには成功、送信に失敗したメールがある場合は別途webhook等で結果を受け取って再処理する必要がある
- 外部APIのレートリミットの考慮が必要

## キューを挟む

メール送信の非同期化を考えます。  
状態の変更さえ行えれば、メールはある程度の時間の枠内で送信できれば良い(例えば、ある時間から30分以内、など)
場合は多いはずです。  
この場合、Amazon SQSなどのキューを使用して、メール送信処理はキューを処理するワーカーに任せる方法があります。  
ワーカーをスケールアウトさせることで高速化も実現できます。

```JavaScript
function get_mails() {
}

function send_mail_by_queue(mails) {
}

function begin() {
}

function update(mails) {
}

function commit() {
}

function rollback() {
}

function notify(errors) {
}

const mails = get_mails()

const errors = [];
try {
    begin()
    update(mails)
    send_mail_by_queue(mails)
    commit()
} catch (e) {
    rollback()
    errors.push(...mails)
}

notify(errors)
```

### pros

- 一般に、キューへの挿入は外部APIの呼び出しに比べて失敗しづらい
- Dead letter queueなどの仕組みでメール送信失敗時に個別にリトライが可能

### cons

- キューを処理するワーカーサーバが別途必要

## まとめ

どの方法を選ぶかは、どのくらいの時間で宛先にメールを届けたいかという要件と、掛けて良いコストに依存します。  
数件から数百件くらいなら、1件ずつ送信する方法が良いでしょう。  
件数が増えてきた場合、かつ、ある程度の時間の間にメールを届けたい場合どうするかを考慮します。  
また、モバイルアプリへのプッシュ通知でもメール送信と同じような問題がありますが、受信した宛先側の反応がメールより早いので、スパイクアクセス対策も考えないといけません。  

と、思いついた方法はこれくらいです。  
もっと良い方法があれば知りたいところですが、この辺の知識って実運用のコード以外ではどこで学ぶのが良いのでしょうか...?
