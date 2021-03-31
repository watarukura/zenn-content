---
title: "DynamoDB テーブルの設計サンプル"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [dynamodb]
published: true
---

# What's

管理画面からDynamoDBテーブルにコンテンツを登録して、APIサーバがレコードを読み出してJSONレスポンスを作るようなケースでよく使うDynamoDBテーブルの設計について書く。

## サンプル

```yaml
TableName: news_release
PartitionKey: id(S)
GSI:
    PartitionKey: display_on(BOOL)
    SortKey: display_end_datetime(N)
```

- 管理画面からコーポレートサイトで表示するニュースリリースを登録する
- 表示・非表示を切り替えたい
- 開始日時から表示を開始、終了日時に表示を終了したい
- 表示終了したレコードは順次削除されてほしい

## 管理画面

- news_releaseテーブルへのput、updateを行なう
- ニュースリリース本文: contents(S)がそれほど長くなければレコードに持ってもよい
  - 1レコードの最大サイズは1KB
  - 1KBを超える場合はURLを保持して別ページに遷移することを考える
- 新規登録時のIDはユニークであれば何でも良い
  - UUIDでも良いし、連番でも良い
  - PartitionKeyが分散しているほうが性能は出るので、リクエストが多いなら、 UUIDなど値がバラけるものを使う
  - news_release程度であれば、それほど頻繁であるとは考えにくいので連番で十分
    - 同一レコードに対する更新も多くないと思われるので、リクエストが多いなら別途キャッシュを考えるほうが良さそう
- display_start_datetime(N)、display_end_datetime(N)は年月日時分秒の精度で登録する
  - 管理画面上は時単位・分単位までしか入力できないようにしておいても良い
- display_on(BOOL) はfalseをdefaultにする
  - 担当部署内でレビューが終わったらtrueにする
  - 公開後に誤りが見つかった場合など、急いで非表示にしたいとき、falseに変更する
- display_end_datetime(N) のnヶ月後をttl(N)として設定する
  - ttl(N)はunixtimeで指定する必要がある

## API

- news_releaseからGSIを使ってqueryする
- display_on(BOOL)がtrue、かつ現在時刻 < display_end_datetime(N)なレコードを取得する
- 現在時刻 > display_start_datetime(N)なレコードをfilter expressionでフィルタすれば必要なレコードが取得できる
  - filter expressionでは取得後のレコードから必要な分だけを選別するので、キー条件式でいかに取得対象を減らせるかが重要
  - display_start_datetime(N)をGSIキーにしたほうが見た目はわかりやすいが、現在時刻は当然のことながら大きくなり続けるので、display_end_datetime(N) をGSIキーにするべき
- display_start_datetime(N) をキーに降順でsortする
- ↓はPartiQLで取得する例

```partiql
SELECT *
FROM news_release
WHERE display_end_datetime > ?
  AND display_start_datetime < ?
```

## まとめ

- DynamoDBのテーブル設計にはあまり実際に使うにあたっての指針になるドキュメントを見つけられていない
- Blackbeltを見たりはしているが...
- いい感じに設計情報がどこかに共有されると嬉しい
