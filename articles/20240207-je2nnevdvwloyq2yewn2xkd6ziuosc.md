---
title: "BigQuery OmniでS3出力されたCURへクエリを投げたい"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud", "AWS", "BigQuery"]
published: true
---

AWSでマルチアカウント運用しているとコスト管理がちょっとたいへんですよね。  
Cost Explorerでは単一アカウントしか見られないので、ほかのアカウントと比べて何にコストが掛かっているかみたいなのはグラフを並べて見られるとうれしい。  
一方で、毎日コストを見ているわけにもいかないので、異常時だけ通知してほしい。
ということで、こんな感じ。terraformで作ります。

```mermaid
graph LR
    subgraph aws 
        subgraph s3 
            CUR -->|scheduled output| CUR
        end
        CUR -->|crawl| Glue
        CostAnomalyDetection
    end
    subgraph googlecloud 
        BigQuery -->|BigQueryOmni| Glue
        Looker --> BigQuery
    end
    CostAnomalyDetection -->|ChatBot| Slack
```

## Cost Anomaly Detection

まずはコスト異常検出です。  
有効化するだけなのでそれほど面倒ではないのですが、AWS Chatbotがterraform対応してないので一工夫します。  
hashicorp/awsccを使うか、aws_cloudformation_stackリソースを使えばよいのですが、CURでCloudFormationテンプレートの読み込みをするので合わせます。

と、解説を書こうと思ったらクラメソさんがすでに書いているのでコレをそのまま使わせていただきます。  
いつもお世話になっています。多謝。  
[AWS Cost Anomaly Detection を Chatbot で Slack 通知させる by Terraform | DevelopersIO](https://dev.classmethod.jp/articles/cost-anomaly-detection-slack-terraform/)

## CUR

CURはus-east-1限定なのでご注意ください。  
CURを出力するS3バケットも同様です。  
BigQueryOmniからはAWS Glueに対してクエリを投げるので、Athena統合を指定します。  
S3のバケットポリシーも適切に設定する必要があります。  
マネジメントコンソールからCURを定義するとバケットポリシーを上書きしてくれるので、これをterraformに取り込んであげると簡単です。

```hcl
resource "aws_cur_report_definition" "example_cur_report_definition" {
  report_name                = "example-cur-report-definition"
  time_unit                  = "HOURLY"
  format                     = "Parquet"
  compression                = "Parquet"
  additional_schema_elements = ["RESOURCES"]
  s3_bucket                  = "example-bucket-name"
  s3_region                  = "us-east-1"
  additional_artifacts       = ["ATHENA"]
  report_versioning          = "OVERWRITE_REPORT"
}

resource "aws_s3_bucket_policy" "cur_bucket_policy" {
  bucket = aws_s3_bucket.cur.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = [
            "bcm-data-exports.amazonaws.com",
            "billingreports.amazonaws.com"
          ]
        }
        Action = [
          "s3:GetBucketPolicy",
          "s3:PutObject",
        ],
        Resource = [
          aws_s3_bucket.cur.arn,
          "${aws_s3_bucket.cur.arn}/*",
        ]
        Condition = {
          StringLike = {
            "aws:SourceArn" = [
              "arn:aws:cur:us-east-1:${local.account_id}:definition/*",
              "arn:aws:bcm-data-exports:us-east-1:${local.account_id}:export/*",
            ]
          }
          StringEquals = {
            "aws:SourceAccount" = local.account_id
          }
        }
      }
    ]
  })
}

```

さて、これをapplyしても、S3にParquet形式で出力されるだけでAthenaでクエリできません。  
まずは、CURの出力を待つ必要があります。1日2回出力されるので、↑を実行して、ステータスがhealthyであることを確認したら翌日まで待ちましょう。

<!-- textlint-disable -->
CURが出力されたら、[AWS CloudFormation テンプレートを使用した Athena のセットアップ - AWS コストと使用状況レポート](https://docs.aws.amazon.com/ja_jp/cur/latest/userguide/use-athena-cf.html)に記載の通り、CloudFormationテンプレートがCURと同じバケットに出力されるので、これを実行します。
<!-- textlint-enable -->

```hcl
resource "aws_cloudformation_stack" "cur_athena" {
  name = "cur-athena"

  template_url = "https://${aws_s3_bucket.cur.bucket}.s3.amazonaws.com/${local.s3_prefix}/${local.account_id}-${local.s3_prefix}/crawler-cfn.yml"
  capabilities = ["CAPABILITY_IAM"]
}
```

## BigQuery Omni

最後にBigQueryです。

```hcl
resource "google_bigquery_connection" "cur" {
  connection_id = local.connection_name
  location      = "aws-us-east-1"
  aws {
    access_role {
      iam_role_id = "arn:aws:iam::${local.account_id}:role/${local.role_name}"
    }
  }
}

resource "google_bigquery_dataset" "cur" {
  provider   = google-beta
  dataset_id = "cur_${local.account_id}_dataset"
  location   = "aws-us-east-1"

  external_dataset_reference {
    external_source = "aws-glue://arn:aws:glue:us-east-1:${local.account_id}:database/athenacurcfn_${local.account_id}_cost_report"
    connection      = "projects/${local.project_name}/locations/aws-us-east-1/connections/${local.connection_name}"
  }
}
```

BigQueryで連携データセットを作ると、BiqQuery Google IDが生成されるので、これを使ってIAMロールを作成します。

```hcl
resource "aws_iam_role" "bigquery" {
  name                 = local.role_name
  assume_role_policy   = data.aws_iam_policy_document.bigquery.json
  max_session_duration = 12 * 60 * 60 # 12h
}

data "aws_iam_policy_document" "bigquery" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = ["accounts.google.com"]
    }
    condition {
      test     = "StringEquals"
      variable = "accounts.google.com:sub"
      values   = [google_bigquery_connection.cur.aws[0].access_role[0].identity]
    }
  }
}

resource "aws_iam_policy" "bigquery" {
  name = "bigquery-external-connection-policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:ListBucket"]
        Resource = aws_s3_bucket.cur.arn
      },
      {
        Effect = "Allow"
        Action = ["s3:GetObject"]
        Resource = [
          aws_s3_bucket.cur.arn,
          "${aws_s3_bucket.cur.arn}/*",
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "glue:GetDatabase",
          "glue:GetTable",
          "glue:GetTables",
          "glue:GetPartitions"
        ]
        Resource = [
          "arn:aws:glue:us-east-1:${local.account_id}:catalog",
          "arn:aws:glue:us-east-1:${local.account_id}:database/athenacurcfn_${local.account_id}_cost_report",
          "arn:aws:glue:us-east-1:${local.account_id}:table/athenacurcfn_${local.account_id}_cost_report/*"
        ]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "bigquery" {
  policy_arn = aws_iam_policy.bigquery.arn
  role       = aws_iam_role.bigquery.name
}
```

これで完成です。  
あとは、LookerなどでBigQueryにクエリして、勝手に更新されてドリルダウンもでき、複数アカウントを並べてチェックできるダッシュボードのできあがりです。

## まとめ

1日待つ、みたいな運用ハックがあるので手間はかかりますが、CloudFormationで作ったものをterraformにimportすればもうちょっと楽に作れるかもしれません。  
このあたりはお好みで。  
コスト異常検出、CUR、BigQueryをそれぞれモジュールにしてアカウントごとにモジュールを呼ぶ、みたいな作りにすると簡単に複数アカウントに対応できます。
