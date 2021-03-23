---
title: "DynamoDB テーブルの設計サンプル"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [dynamodb]
published: false
---

# What's

管理画面から DynamoDB テーブルにコンテンツを登録して、API サーバがレコードを読み出して JSON レスポンスを作るようなケースでよく使う DynamoDB テーブルの設計について書く。

## サンプル

TableName: news_release
PartitionKey: id(S)
GSI:
    PartitionKey: display_on(BOOL)
    SortKey: display_end_datetime(N)

- 管理画面からコーポレートサイトで表示するニュースリリースを登録する
- 表示・非表示を切り替えたい
- 開始日時から表示を開始、終了日時に表示を終了したい
- 表示終了したレコードは順次削除されてほしい

## 管理画面

- news_release テーブルへの put、update を行う
- ニュースリリース本文: contents(S)がそれほど長くなければレコードに持ってもよい
    - 1 レコードの最大サイズは 1KB
    - 1KB を超える場合は URL を保持して別ページに遷移する事を考える
- 新規登録時の ID はユニークであれば何でも良い
    - UUID でも良いし、連番でも良い
    - PartitionKey が分散している方が性能は出るので、リクエストが多いなら、 UUID など値がバラけるものを使う
    - news_release 程度であれば、それほど頻繁であるとは考えにくいので連番で十分
        - 同一レコードに対する更新も多くないと思われるので、リクエストが多いなら別途キャッシュを考えるほうが良さそう
- display_start_datetime(N)、display_end_datetime(N)は年月日時分秒の精度で登録する
    - 管理画面上は時単位・分単位までしか入力できないようにしておいても良い
- display_on(BOOL) は false を default にする
    - 担当部署内でレビューが終わったら true にする
    - 公開後に誤りが見つかった場合など、急いで非表示にしたいとき、false に変更する
- display_end_time(N) の n ヶ月後を ttl(N)として設定する
    - ttl(N)は unixtime でしているする必要がある

## API

- news_release から GSI を使って query する
- display_on(BOOL)が true、かつ現在時刻 < display_end_datetime(N)なレコードを取得する
- 現在時刻 > display_start_datetime(N)なレコードを filter expression でフィルタすれば必要なレコードが取得できる
    - filter expression では取得後のレコードから必要な分だけを選別するので、キー条件式でいかに取得対象を減らせるかが重要
    - display_start_datetime(N)を GSI キーにした方が見た目はわかりやすいが、現在時刻は当然のことながら大きくなり続けるので、display_end_datetime(N) を GSI キーにするべき
- display_start_datetime(N) をキーに降順で sort する

```partiql
SELECT *
FROM news_release
WHERE display_end_datetime > ?
  AND display_start_datetime < ?
```

## まとめ

- DynamoDB のテーブル設計にはあまり実際に使うにあたっての指針になるドキュメントを見つけられていない
- Blackbelt を見たりはしているが...
- いい感じに設計情報がどこかに共有されると嬉しい