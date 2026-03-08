---
title: "terraform plan -generate-config-out後のコードを簡素にする"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform"]
published: true
---

IaCとしてterraformを導入する際、`terraform plan -generate-config-out=out.tf` はとても便利です。  
<https://developer.hashicorp.com/terraform/language/import/generating-configuration>

ただ、実行後に属性がズラッと並んで冗長であること、他リソースとの関連付けがされず文字列になってしまうところを1件ずつ関連付けしていかないとメンテナンスが困難になります。  
この記事では、前者を解決するツールを開発したので紹介します。  
(後者については今のところ手作業でどうにかする必要があります...。)

## tfsimplify

[tfsimplify](https://github.com/watarukura/tfsimplify) はterraform providerの各属性についてデフォルト値と同じ行を削除するツールです。

```bash
Usage: tfsimplify -dir <directory> [options]

tfsimplify removes attributes from Terraform .tf files that match
the provider schema's default values.

Options:
  -check
        exit 1 if changes are needed (no write)
  -dir string
        directory to scan for .tf files (required)
  -write
        rewrite files in-place

Examples:
  tfsimplify -dir ./environments/prod
  tfsimplify -dir ./environments/prod -write
  tfsimplify -dir ./environments/prod -check
```

以下の様なterraformコードを想定します。

```bash
❯ ls
s3.tf  terraform.tf
❯ terraform init
Initializing the backend...
# 後略
❯ ls -la
drwxr-xr-x 10 watarukura staff  320 Mar  7 20:01 .
drwxr-xr-x 21 watarukura staff  672 Mar  8 11:01 ..
drwxr-xr-x  4 watarukura staff  128 Mar  7 11:58 .terraform
-rw-r--r--  1 watarukura staff 1406 Mar  7 11:05 .terraform.lock.hcl
-rw-r--r--  1 watarukura staff  201 Mar  7 20:01 s3.tf
-rw-r--r--  1 watarukura staff  195 Mar  7 11:05 terraform.tf
```

```hcl
resource "aws_s3_bucket" "example" {
  bucket              = "example"
  bucket_prefix       = null
  force_destroy       = false
  object_lock_enabled = false
  tags                = {}
}
```

`-check/--check` オプションを付与すると、unified形式のdiffで削除する箇所を表示します。  
差異がある場合はexit status 1で終了するので、CIなどでご利用ください。

```bash
❯ tfsimplify -check --dir .
--- s3.tf
+++ s3.tf
@@ -1,8 +1,6 @@
 resource "aws_s3_bucket" "example" {
   bucket              = "example"
   bucket_prefix       = null
-  force_destroy       = false
   object_lock_enabled = false
-  tags                = {}
 }

exit status 1
```

`-write/--write` オプションを付与すると、実際に削除します。

```bash
❯ tfsimplify -write -dir .
updated: s3.tf

❯ diff -u s3.tf.org s3.tf
--- s3.tf.org   2026-03-08 11:26:51.587601329 +0900
+++ s3.tf       2026-03-08 11:27:04.515600146 +0900
@@ -1,8 +1,6 @@
 resource "aws_s3_bucket" "example" {
   bucket              = "example"
   bucket_prefix       = null
-  force_destroy       = false
   object_lock_enabled = false
-  tags                = {}
 }
```

`-check/--check`と`-write/--write`オプションは同時に指定できません。
また、いずれのオプションも指定しない場合は、更新対象ファイル名のみ出力して正常終了します。

```bash
❯ tfsimplify --dir .
would change: s3.tf
```

## 開発方法

フルにVibe Codingしました。  
まずは、思いついた内容をGPT-5.2に相談します。  
`terraformコードのlinterを自作しようと思います。各attributeについて、defaultの値と同じだった場合は当該行を削除する、というルールにしたいです。`.
↓回答の一部を抜粋します。

```markdown
## 1) 実装方針（おすすめ）

### A. “default” の情報源は provider schema を使う（本命）

Terraform の default は「HCLの型だけ」では決まりません。ほとんどは **provider 側の resource schema** に入っています。

**実装イメージ**

1. 対象ディレクトリで `terraform init`（CI なら事前にやる）
2. `terraform providers schema -json` を実行して **resource/data source の schema を JSON で取得**
3. HCL をパースし、各 `resource "X" "Y" { ... }` を見つける
4. schema からその resource type の attribute を引き、`Optional` & `Default` を参照
5. HCL 側の attribute が **リテラル（文字列/数値/bool/null/list/map/object）** で、かつ schema default と同値なら削除
6. `hclwrite` で整形しつつファイルに書き戻す（または diff を出す）

> これが一番 “Terraform の真実” に寄ります。

---

## 2) 具体的な実装構成（Go 例が相性良い）

Terraform/HCL 周りのライブラリが強いので Go が一番ラクです。

### 使うもの

* `hashicorp/hcl/v2` + `hclparse`（読む）
* `hashicorp/hcl/v2/hclwrite`（編集して書く）
* `terraform providers schema -json`（schema取得）

### 削除アルゴリズム（安全寄り）

* まず「削除してよい」条件をかなり絞るのがコツです

  * attribute 値が **式じゃなくてリテラル**（`true`, `"abc"`, `123`, `[]`, `{}` など）
  * schema default も JSON で確定値として取れる
  * “差分なし”の同値判定ができる（後述）
* その上で一致したら `hclwrite.Body().RemoveAttribute(name)` で消す

```

なるほど、`terraform providers schema -json`コマンドでスキーマ情報がJSON形式で取得できるんですね。初耳。
HCLのparse用のライブラリを使うためにGoで書くのも納得できる判断なので採用します。

最初はAWSプロバイダーのみとして、小さく実装する方針で、初期のコードおよび設計書をmarkdownで書いてもらいました。  
これをmain.go、docs/design.mdに書き出します。  
`go mod tidy` の実行くらいは人の手でやりました。
あとはAI Agentに実際のコードを書いてもらいます。

- AI Agent: Junie version 888.57
- Model: Claude Opus 4.6

以下、Junieとのやり取りの抜粋です。
`~/.junie/history.jsonl` にチャットの履歴が出力されます。

```json lines
{"text":"@docs/design.md を参照して、テストコードを追加してください","timestamp":1772788200456,"projectPath":"/Users/watarukura/src/github.com/watarukura/tfsimplify"}
```

できあがったコマンドを実行しても想定の挙動にならない上、テストは全部通っているので実際に実行してみて再度Junieに聞いたりしました。

```json lines
{"text":"go run ../main.go -check . をカレントディレクトリで実行しても正常終了します。","timestamp":1772847035714,"projectPath":"/Users/watarukura/src/github.com/watarukura/tfsimplify/example"}
```

動くようになったら、機能追加です。

```json lines
{"text":"`# tfsimplify-disable` と `# tfsimplify-enable` で挟まれた行を無視する、`# tfsimplify-ignore` の直下の行を無視するルールにしたいです。testにも反映してください","timestamp":1772881393040,"projectPath":"/Users/watarukura/src/github.com/watarukura/tfsimplify"}
{"text":"terraform-provider-aws 以外のterraformプロバイダーにも対応したい","timestamp":1772935092436,"projectPath":"/Users/watarukura/src/github.com/watarukura/tfsimplify"}
```

最後にテストカバレッジもいい感じに。

```json lines
{"text":"❯ go test -cover ./...\nok      github.com/watarukura/tfsimplify        0.540s  coverage: 49.7% of statements\nテストカバレッジを80%以上にしてください","timestamp":1772938659984,"projectPath":"/Users/watarukura/src/github.com/watarukura/tfsimplify"}
```

## まとめ

[Linter 開発こそ Vibe Coding の最も優れた適用例の 1 つである](https://zenn.dev/yumemi_inc/articles/go-linter-vibe-coding)
mpywさんがLinter作るといいぞ、という話をされてたので影響を受けて作ってみました。

ほぼほぼコードを書くことなく希望のツールが完成し、README.mdも少し手直しした程度です。  
思想が強めのツールなので好みはあると思いますが、[tfsimplify](https://github.com/watarukura/tfsimplify)、よかったら使ってみてください。
