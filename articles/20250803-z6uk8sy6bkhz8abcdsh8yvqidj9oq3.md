---
title: "terraform local moduleの更新差分をterraform-config-inspectで抽出する"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform"]
published: true
---

<https://github.com/suzuki-shunsuke/tfactionn> のコードを読んで、moduleの更新差分の取得部分(js/src/list-module-callers/index.ts)をシェルスクリプトで再実装して理解を検証する、みたいな内容です。  
もろに車輪の再発明で特に実用性はないですが、ちょっと面白かったので。
前職で公開しているOSSの<https://github.com/torana-us/tfdir>の挙動の理解には役立ちました。

## Why?

以下の様なterraform用のリポジトリがあったとします。
iam moduleはapp moduleから呼び出すローカルmoduleです。

```shell
.
|-- environments
|   `--prd
|     |--app     # applicationの設定(EC2など)
|     `--network # networkの設定(VPC、subnetなど)
|   `--stg
|     |--app
|     `--network
`-- modules
    |--app
    |--iam       # appが利用しているIAM
    `--network
```

prd環境とstg環境へのplan/applyはそれぞれ別のタイミングで実行したいです。  
また、prd環境分だけ更新したい場合にstg環境のplan/applyが動作してほしくないし、prd/appの更新時にprd/networkはplan/applyされてほしくありません。  
このとき、environmentsディレクトリ以下の更新については`git diff`で更新のあったディレクトリだけ対象にすれば良いです。  
しかし、modulesディレクトリ以下が更新された場合は、更新されたmoduleを呼んでいるenvironmentsだけをplan/apply対象にしたいです。

## hashicorp/terraform-config-inspect

<https://github.com/hashicorp/terraform-config-inspect> はaquaproj/aquaでinstallするのが簡単です。  
以下のaqua.ymlがサンプルです。  
GitHub Releasesがないので、versionにはcommit ハッシュを指定します。

```yaml
---
# yaml-language-server: $schema=https://raw.githubusercontent.com/aquaproj/aqua/main/json-schema/aqua-yaml.json
# aqua - Declarative CLI Version Manager
# https://aquaproj.github.io/
# checksum:
#   enabled: true
#   require_checksum: true
#   supported_envs:
#   - all
registries:
- type: standard
  ref: v4.397.0 # renovate: depName=aquaproj/aqua-registry
  packages:
- name: hashicorp/terraform-config-inspect
  version: "e8a84eebd3e740ab8dc37d5c8f843523e7d49e8d"
```

↓コレが出力です。

```json
❯ terraform-config-inspect --json environments/prd/network | jq -r '.module_calls'
{
  "network": {
    "name": "network",
    "source": "../../../modules/network",
    "pos": {
      "filename": "environments/prd/network/network.tf",
      "line": 1
    }
  }
}
```

module呼び出し側から見て、moduleのディレクトリがどこにあるか、がわかります。  
ほしいのは、module側から見て呼び出し側のディレクトリがどこにあるか、なので、ひと工夫要ります。  

```shell
# 1. terraformコードを洗い出す
❯ find ./environments -type f -name "*.tf" \
    # 2. ディレクトリの一覧を取得
    | cut -d'/' -f2,3,4 \
    # 3. 重複を削除
    | sort -u \
    # 4. ディレクトリごとにterraform-config-inspectを実行し、呼び出しているmoduleのうちローカルmoduleのみに絞ってリストアップして、呼び出しディレクトリとmoduleディレクトリを紐づける
    | xargs -I{} bash -c "terraform-config-inspect --json {} | jq -r '.module_calls | flatten | .[].source' | sort -u | grep '^../' | sed -e 's;../../../;;' | awk -v dir={} '{print dir, \$1}'"
        jq -r '.module_calls // {} | to_entries[] | .value.source | select(startswith("./") or startswith("../"))' "$temp_output" >module_map 2>/dev/null || true

❯ cat module_map
environments/prd/app modules/app
environments/prd/network modules/network
environments/stg/app modules/app
environments/stg/network modules/network
```

tfactionでは、moduleからネストしてmoduleを呼んでいる場合にも対応しています。  

```shell
❯ find ./modules -type f -name "*.tf" \
    | cut -d'/' -f2,3 \
    | sort -u \
    | xargs -I{} bash -c "terraform-config-inspect --json {} | jq -r '.module_calls | flatten | .[].source' | sort -u | grep '^../' | sed -e 's;../;modules/;' | awk -v dir={} '{print dir, \$1}'" \
    >module_called_module

❯ cat module_called_module
modules/app modules/iam

❯ while read -r calling called; do
    awk -v calling="$calling" -v called="$called" \
      '$2==calling{print $1, called}' module_map >>nested_module
  done <module_called_module

❯ cat module_map nested_module
environments/prd/app modules/app
environments/prd/network modules/network
environments/stg/app modules/app
environments/stg/network modules/network
environments/prd/app modules/iam
environments/stg/app modules/iam
```

上記をひとまとめにして、list-called-module.bashとして保存します。

```shell
#!/usr/bin/env bash
set -euo pipefail
find ./environments -type f -name "*.tf" \
  # 2. ディレクトリの一覧を取得
  | cut -d'/' -f2,3,4 \
  # 3. 重複を削除
  | sort -u \
  # 4. ディレクトリごとにterraform-config-inspectを実行し、呼び出しているmoduleのうちローカルmoduleのみに絞ってリストアップして、呼び出しディレクトリとmoduleディレクトリを紐づける
  | xargs -I{} bash -c "terraform-config-inspect --json {} | jq -r '.module_calls | flatten | .[].source' | sort -u | grep '^../' | sed -e 's;../../../;;' | awk -v dir={} '{print dir, \$1}'"
      jq -r '.module_calls // {} | to_entries[] | .value.source | select(startswith("./") or startswith("../"))' "$temp_output" >module_map 2>/dev/null || true
find ./modules -type f -name "*.tf" \
  | cut -d'/' -f2,3 \
  | sort -u \
  | xargs -I{} bash -c "terraform-config-inspect --json {} | jq -r '.module_calls | flatten | .[].source' | sort -u | grep '^../' | sed -e 's;../;modules/;' | awk -v dir={} '{print dir, \$1}'" \
  >module_called_module
while read -r calling called; do
  awk -v calling="$calling" -v called="$called" \
    '$2==calling{print $1, called}' module_map >>nested_module
done <module_called_module
```

## GitHub Actionsへの組み込み

module_map / nested_moduleファイルを使って、更新差分がある場合だけplan / applyするようにします。  
Gitの更新差分を取得して、module_map / nestd_moduleファイルと突き合わせてplanするディレクトリを決定します。  
以下はAWSへの例ですが、他のCloud Providerでも特段やり方は変わらないです。

```yaml
name: terraform plan stg
on:
  pull_request:
    branches:
      - stg
    paths:
      - '*.tf'
jobs:
  terraform-dir:
    name: Get diff dirs
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    timeout-minutes: 3
    outputs:
      diff-dirs: ${{ steps.diff-dirs.outputs.diff-dirs }}
    steps:
      - name: Checkout
        uses: actions/checkout@692973e3d937129bcbf40652eb9f2f61becf3332 # v4
        with:
          fetch-depth: 0
      - uses: aquaproj/aqua-installer@6ce1f8848ec8e61f14d57bd5d7597057a6dd187c # v3.0.1
        with:
          aqua_version: v2.30.0
      - name: Gen module map
        id: module-map
        shell: bash
        run: |
          ./list-called-module.bash
          cat nested_module >> module_map
      - name: Get diff dirs
        id: diff-dirs
        shell: bash
        run:
          if [[ $(git diff \
                    --diff-filter=d \
                    --name-only origin/${{ github.base_ref }} HEAD \
                    -- modules \
                    | wc -l) -ne 0 ]]; then
            # modules配下に差分がある場合
            touch module_diff

            git diff \
              --diff-filter=d \
              --name-only origin/${{ github.base_ref }} HEAD \
              -- modules \
              | xargs dirname \
              | sort -u \
              | xargs -I{} bash -c \
                "awk -v dir={} '\$2 == dir{print \$1}' <module_map >> module_diff"
            diff_dirs=$(git diff \
                          --diff-filter=d \
                          --name-only origin/${{ github.base_ref }} HEAD \
                          -- environments \
                          | cut -d'/' -f1,2,3 \
                          | cat - module_diff \
                          | sort -u \
                          | xargs \
                          | tr -d '\n' \
                          | jq -csR 'split(" ")')
          else
            # modules配下には差分がない場合
            diff_dirs=$(git diff \
                          --diff-filter=d \
                          --name-only origin/${{ github.base_ref }} HEAD \
                          -- environments \
                          | cut -d'/' -f1,2,3 \
                          | sort -u \
                          | xargs \
                          | tr -d '\n' \
                          | jq -csR 'split(" ")')
          fi
          echo "diff-dirs=$diff_dirs" >> "$GITHUB_OUTPUT"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  terraform:
    needs: [terraform-dir]
    name: Terraform plan
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    environment:
      name: staging
    strategy:
      matrix:
        dir: ${{ fromJson(needs.terraform-dir.outputs.diff-dirs) }}
    steps:
      - name: Checkout
        timeout-minutes: 3
        uses: actions/checkout@692973e3d937129bcbf40652eb9f2f61becf3332 # v4
        with:
          fetch-depth: 0
      - name: Set up AWS credentials
        timeout-minutes: 3
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY }}
          aws-region: ap-northeast-1
      - uses: aquaproj/aqua-installer@6ce1f8848ec8e61f14d57bd5d7597057a6dd187c # v3.0.1
        timeout-minutes: 3
        with:
          aqua_version: v2.30.0
      - name: Terraform plan
        timeout-minutes: 10
        run: |
          tfcmt -var "target:${{ matrix.dir }}/$env" plan -patch -skip-no-changes -- \
            ./run.sh ${{ matrix.dir }} "$env" plan
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          env: ${{ vars.ENV }}
```

これで、差分だけplanできるようになりました。

## まとめ

素直にtfactionを使いましょう。必要なら、差分検知の部分だけ利用することもできるはず？  
読んでいただけば分かる通り、シェルスクリプト + jqでがんばるのはなかなか大変でした。
