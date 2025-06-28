---
title: "renovateのない環境で諸々まとめてupgradeする"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aquaclivm"]
published: true
---

ものすごくコストセンシティブな局面で、わがままに最新バージョンを使いたい人向けです。  

さて、最新バージョンをきっちり追っかけたいなら[renovate](https://docs.renovatebot.com/)を使えば良いです。  
github.comにコードをホストしてるならGitHub ActionsでCIしていれば、test / lintを継続的に実行できます。  
しかし、お金がない。ものすごくお金がない。  
かつ、面倒を見てるコードはprivateリポジトリにある、そんなニッチな状況に置かれたあなたへの記事です。

## aqua管理下のツールをupgradeする

開発に必要なツール群(formatter / linter / etc...)は[aqua](https://github.com/aquaproj/aqua)でinstallします。

aqua本体のupgradeは、以下の通り。

```shell
aqua upa
```

aqua.yaml記載のツールのupgradeは、以下の通りです。

```shell
aqua up
```

## GitHub Actionsのactionをupgradeする

`aqua i suzuki-shunsuke/pinact` しておきます。

```shell
pinact run -u
```

↓commit ハッシュでバージョン固定される上、バージョン番号がコメントされるので安心です。

```yaml
jobs:
  lint:
    name: textlint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

## tflintプラグインをupgradeする

tflintコマンドでバージョンアップできると嬉しいのですが、残念ながらそんな機能はなさそう。ままならないものです。  
シェル芸でどうにかします。  
`aqua i cli/cli` して、ghコマンドをinstallしておきます。  
以下は、Google Cloudプラグインのサンプルです。

```shell
latest_version=$(gh api \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    /repos/terraform-linters/tflint-ruleset-google/releases/latest \
    --jq .tag_name | tr -d 'v')
gsed -i 's/version = "[0-9\.]*"/version = "'"$latest_version"'"/' .tflint.hcl
```

↓以下のversionの箇所を、GitHub.comの最新releaseのバージョンに書き換えます。

```hcl
plugin "google" {
 enabled = true
 version = "0.33.0"
 source  = "github.com/terraform-linters/tflint-ruleset-google"
}
```

## まとめて実行

では、↑をまとめて1コマンドで実行できるようにしましょう。  
先ほどから申し上げている通り、お金がないのでCIをGitHub Actionsで行う余力がありません。  
(GitHub Actionsの無償枠が余っているならもちろんCIに使ってOKです。  
 余ってないのでCDに使うことにしました。  
 安全にはコストをかけるべき、という判断)  

ローカルでCIを行うために[lefthook](https://github.com/evilmartians/lefthook)を導入します。  
`aqua -i evilmartians/lefthook`します。  
lefthook自体はGit hook管理のツールなのですが、任意のスクリプトを実行することもできます。
<https://github.com/evilmartians/lefthook#your-own-tasks>

以下はPython + terraformプロジェクト用のサンプルです。  
これで、`lefthook run bump`で依存関係をまとめてupgradeできます。

```yaml
pre-commit:
  parallel: true
  commands:
    tffmt:
      glob: "*.{tf,hcl}"
      run: terraform fmt -recursive
    tflint:
      glob: "*.{tf,hcl}"
      run: |
        config_path=$(PWD)/.tflint.hcl
        tflint --config "$config_path" --recursive --init
        tflint --config "$config_path" --recursive
    trivy:
      glob: "*.{tf,hcl}"
      run: |
        trivy fs --scanners secret,misconfig \
          --severity MEDIUM,HIGH,CRITICAL \
          --ignorefile ./.trivyignore \
          --exit-code 1 .
    typos:
      run: typos --config .typos.toml .
    actionlint:
      root: ".github/workflows"
      glob: "*.{yml,yaml}"
      run: actionlint
    yamlfmt:
      glob: "*.{yml,yaml}"
      run: yamlfmt -conf .yamlfmt.yml .
    ruff:
      glob: "*.py"
      run: |
        uv run ruff format
        uv run ruff check --fix
    pytest:
      glob: "*.py"
      run: uv run pytest --cov
bump:
  parallel: true
  commands:
    aqua:
      run: aqua upa && aqua up
    pinact:
      run: pinact run -u
    tflint:
      run: |
        latest_version=$(gh api \
          -H "Accept: application/vnd.github+json" \
          -H "X-GitHub-Api-Version: 2022-11-28" \
          /repos/terraform-linters/tflint-ruleset-google/releases/latest \
          --jq .tag_name | tr -d 'v')
        gsed -i 's/version = "[0-9\.]*"/version = "'"$latest_version"'"/' .tflint.hcl
    python:
      run: uv lock --upgrade && uv sync --upgrade
```
