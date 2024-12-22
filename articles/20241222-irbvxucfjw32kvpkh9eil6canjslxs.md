---
title: "CURをduckdbでクエリしてevidenceでチャートにする"
emoji: "🦆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "duckdb", "evidence"]
published: true
---

[BigQuery OmniでS3出力されたCURへクエリを投げたい](https://zenn.dev/watarukura/articles/20240207-je2nnevdvwloyq2yewn2xkd6ziuosc)  
↑こちらの記事の続きです。
AWSコスト可視化のための環境は2日で作れるようになりました。  
が、BigQuery + Athena + Lookerという環境なので、開くたびにクエリが実行され課金対象になってしまいます。  
コスト削減のためにコストがかかるのも本末転倒感があるな、と思っていたところ、ちょうどduckdbを弄りたかったのでちょうどいいネタになると思い作ってみました。

## CURを取得する

前回の記事では、Cost Usage Report(以下、CUR)をParquet形式で出力しておきました。  
これにduckdbでクエリを投げます。

```sql
CREATE OR REPLACE TABLE cur AS
SELECT line_item_usage_start_date,
       line_item_product_code,
       SUM(line_item_unblended_cost) AS cost
FROM "s3://${account_id}-cur/cost-report/${account_id}-cost-report/${account_id}-cost-report/year=${year}/month=${month}/*.parquet"
GROUP BY 1, 2;
```

S3のパスなどはterraformでの作成時に指定した通りです。  
(この辺り、tfstateから取ってこられると、もっとスマートかもしれません)  
これで、duckdbファイル内のcurテーブルに時間単位・AWSプロダクト単位のコストが集計されました。  
duckdbなので、GROUP BYももっとスマートに書けるのですが手抜きしています。

## evidenceでチャート作成

evidenceをセットアップします。  
<https://github.com/evidence-dev/template#get-started-using-the-cli> こちらの通り実行します。

```bash
npx degit evidence-dev/template my-project
cd my-project 
npm install 
npm run sources
npm run dev 
```

sampleのチャートが表示されることを確認して、ターミナルからCtrl + Cで終了させます。  
duckdbへのクエリ結果をチャートにしたいので、新規のsourceを追加します。

```bash
mkdir -p sources/cur_${account_id}
cp cur.duckdb sources/cur_${account_id}/
echo 'select * from cur' > sources/cur_${account_id}/cur.sql
cp sources/needful_things/connection.yaml sources/cur_${account_id}/
```

sources/cur_${account_id}/connection.yamlをduckdb用に書き換えます。

```yaml
type: duckdb
name: cur_${account_id}
options:
  filename: cur.duckdb
```

sourceの追加・修正の都度、npm run sourcesを実行します。  
pages/index.mdを書き換えて、duckdbへのクエリ結果を出力するようにします。
markdownで書くSQLからは、sourcesファイル中の*.sqlがテーブル扱いになるようです。  
(cur.sqlを置かないと、「そんなテーブルないよ！」ってエラーが出ます)

````markdown
---
title: CUR
---

<BarChart
data={cur_${account_id}}
title="AWS Daily Costs"
x=date
y=cost
series=product
/>

```sql cur_${account_id}
SELECT date_trunc('day', line_item_usage_start_date) AS date,
       line_item_product_code AS product,
       SUM(cost) AS cost
FROM cur_${account_id}.cur
GROUP BY date, product;
```
````

チャートになりました。
![img.png](https://github.com/watarukura/zenn-content/blob/main/articles/images/20241222_evidence.png?raw=true)
時刻単位で集計したらBarが細くなりすぎて見づらいので日毎の集計にしています。

## GitHub Pagesにデプロイ

GHECでprivateなGitHub Pagesが利用できるプランを契約しているので、社内公開してみます。  
リポジトリのSettingsからGitHub Pagesを有効化し、デプロイ用のGitHub Actions workflowを作成します。

```yaml
---
name: Deploy to GitHub Pages
on:
  schedule:
    - cron: "0 0 * * *" # 09:00 毎日 JST
  workflow_dispatch:
jobs:
  fetch:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    steps:
      - name: Checkout
        timeout-minutes: 3
        uses: actions/checkout@v4
      - name: Set up AWS credentials
        timeout-minutes: 3
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.OIDC_ARN }} # OIDC用のarnを指定
          aws-region: us-west-1
      - uses: aquaproj/aqua-installer@v3.1.0
        timeout-minutes: 3
        with:
          aqua_version: v2.40.0
      - name: Update duckdb
        run: |
          ./update_duckdb.bash
      - name: Upload cur.duckdb
        uses: actions/upload-artifact@v4
        with:
          name: cur_${account_id}
          path: sources/cur_${account_id}/cur.duckdb
          retention-days: 1
  deploy:
    needs: fetch
    runs-on: ubuntu-latest
    permissions:
      pages: write
      contents: read
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Checkout
        timeout-minutes: 3
        uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: 'package.json'
      - uses: actions/cache@v4
        with:
          path: '**/node_modules'
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
      - name: Download cur.duckdb
        uses: actions/download-artifact@v4
        with:
          name: cur_${account_id}
          path: sources/cur_${account_id}
      - name: Build
        run: |
          npm i
          npm run sources
          npm run build
      - name: Upload Artifacts
        uses: actions/upload-pages-artifact@v3
        with:
          path: 'build'
      - name: Deploy
        id: deployment
        uses: actions/deploy-pages@v4
```

update_duckdb.bashの中身は↓これです。  
原因は調査中なのですが、duckdb経由でのS3へのアクセスで400エラーが出るため、parquetファイルをダウンロードして読み込むようにしました。

```bash
#!/usr/bin/env bash
set -euo pipefail
set -xv
account_id=$(aws sts get-caller-identity --query Account --output text --no-cli-pager)
cd "sources/cur_${account_id}/"
rm -f cur.duckdb
aws s3 cp s3://${account_id}-cur/cost-report/${account_id}-cost-report/${account_id}-cost-report/ . --recursive

duckdb cur.duckdb <<SQL
CREATE OR REPLACE TABLE cur AS
SELECT line_item_usage_start_date,
       line_item_product_code,
       SUM(line_item_unblended_cost) AS cost
FROM "./**/*.parquet"
GROUP BY 1, 2;
SQL
```

また、上記は1アカウントでの結果出力ですが、複数アカウントでの結果を集計して1ページに出力したくてupload/download-artifact actionを使っています。

## まとめ

<!-- textlint-disable -->
duckdbもevidenceもいいぞ！

https://github.com/duckdb/duckdb

https://github.com/evidence-dev/evidence
<!-- textlint-enable -->
