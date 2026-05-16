---
title: "Bedrockコストを利用者別に集計する"
emoji: ":moneybag:"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Bedrock"]
published: true
---

[IAM プリンシパルベースのコスト配分で Amazon Bedrock のコストを呼び出し元ごとに追跡する | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/track-amazon-bedrock-costs-by-caller-identity-with-iam-based-cost-allocation/)
Bedrockのコストを利用者別に集計するのに、従来はアプリケーション推論プロファイルを使用していましたが、IAMプリンシパルベースのコスト配分でAmazon Bedrockのコストを呼び出し元ごとに追跡する方法が提供されました。
アプリケーション推論プロファイルをClaudeの新モデルが出る度に作成して配布するのは地味に面倒だったので、大変ありがたいです。
ということで、terraformでの実装も含めてご紹介。

なお、アプリケーション推論プロファイルを使う場合の設定は、こちらの記事が参考になりました。
[Bedrock を組織で使う場合に推論プロファイル経由のみ許可する権限設計について考える](https://zenn.dev/winteryukky/articles/b51ba6d63cf15e)
Bedrock + Claude Code ActionsなどでGitHub Actionsから利用する場合などは設定しておいても良いかもしれません。

## 前提

- IAM Identity Centerを設定してSSOログインを有効化済みであること

## CUR2.0を設定する

CUR2.0および出力先のS3バケットはus-east-1に作成する必要があるので注意してください。  

```hcl
resource "aws_bcmdataexports_export" "cur" {
  export {
    name = "cur2_0"
    data_query {
      query_statement = "SELECT line_item_unblended_cost, line_item_usage_start_date, product, line_item_iam_principal FROM COST_AND_USAGE_REPORT"
      table_configurations = {
        COST_AND_USAGE_REPORT = {
          BILLING_VIEW_ARN                      = "arn:aws:billing::${data.aws_caller_identity.self.account_id}:billingview/primary"
          INCLUDE_IAM_PRINCIPAL_DATA            = "TRUE"
          INCLUDE_CAPACITY_RESERVATION_DATA     = "FALSE"
          INCLUDE_MANUAL_DISCOUNT_COMPATIBILITY = "FALSE"
          INCLUDE_RESOURCES                     = "FALSE"
          INCLUDE_SPLIT_COST_ALLOCATION_DATA    = "FALSE"
          TIME_GRANULARITY                      = "HOURLY"
        }
      }
    }
    destination_configurations {
      s3_destination {
        s3_bucket = aws_s3_bucket.cur.bucket
        s3_prefix = "cur2_0"
        s3_region = "us-east-1"
        s3_output_configurations {
          compression = "PARQUET"
          format      = "PARQUET"
          output_type = "CUSTOM"
          overwrite   = "OVERWRITE_REPORT"
        }
      }
    }
    refresh_cadence {
      frequency = "SYNCHRONOUS"
    }
  }
}
```

上記を設定すると、CUR2.0が有効になり、us-east-1のS3バケットにデータが定期的に出力されます。  
日次でParquetファイルを上書き出力、100以上あるカラムから必要なものだけに絞っています。  
`INCLUDE_IAM_PRINCIPAL_DATA` が重要です。これを`"TRUE"`に指定することで、IAMプリンシパルベースの集計ができるようになるため。  
初回出力は最大24時間かかる場合があるので、作成したら翌日まで待ってください。

## duckdbで集計する

Claude以外のモデルを使用している場合は、WHEREの箇所は削除してしまって良いです。
line_item_iam_principalから、メールアドレスだけ取り出したいので `split()` を使用しています。

```sql
❯ aws-vault exec example-aws.ReadonlyAccess --region=us-east-1 -- duckdb              
Opening the SSO authorization page in your default browser (use Ctrl-C to abort)
DuckDB v1.5.2 (Variegata)
Enter ".help" for usage hints.
memory D INSTALL httpfs;
memory D LOAD httpfs;
memory D SELECT line_item_usage_start_date,
                COALESCE(
                        split(line_item_iam_principal, '/')[-1],
                        'unknown'
                ) AS user,
                SUM(line_item_unblended_cost) AS cost
         FROM read_parquet("s3://${S3_BUCKET}/**/cur2_0*.parquet")
         WHERE product.product_name LIKE 'Claude%'
         GROUP BY 1, 2
         HAVING SUM(line_item_unblended_cost) > 0.0
         ORDER BY 1, 2;

┌────────────────────────────┬──────────────────────────────┬─────────────────────┐
│ line_item_usage_start_date │             user             │        cost         │
│         timestamp          │           varchar            │       double        │
├────────────────────────────┼──────────────────────────────┼─────────────────────┤
│ 2026-04-01 03:00:00        │ unknown                      │          0.11111111 │
│ 2026-04-01 07:00:00        │ unknown                      │          0.44444444 │
│ 2026-04-01 08:00:00        │ unknown                      │         0.777777777 │
│          ·                 │    ·                         │           ·         │
│          ·                 │    ·                         │           ·         │
│          ·                 │    ·                         │           ·         │
│ 2026-05-15 10:00:00        │ hoge@example.jp              │         0.555555555 │
│ 2026-05-15 11:00:00        │ fuga@example.jp              │          1.11111111 │
│ 2026-05-15 11:00:00        │ piyo@example.jp              │          0.77777777 │
└────────────────────────────┴──────────────────────────────┴─────────────────────┘
```

2026-04-08まではIAMプリンシパルが未設定だったため、それ以前のデータは利用者が`unknown`になります。

## evidence-dev/evidenceでダッシュボードを作成する

[CURをduckdbでクエリしてevidenceでチャートにする](https://zenn.dev/watarukura/articles/20241222-irbvxucfjw32kvpkh9eil6canjslxs)
設定方法は↑上の記事で書きました。

![img.png](images/20260516_bedrock_cost.png)

## まとめ

これで、誰が・いつ・いくら分・どのモデルを呼び出したか可視化できるようになりました！  
DuckDBのクエリ結果をJSON出力してLLMに読み込ませてレポートを作成、とかもできますね。
