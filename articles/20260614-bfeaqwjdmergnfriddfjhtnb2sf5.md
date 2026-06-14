---
title: "aqua checksumを試す"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aqua"]
published: true
---

GitHub ActionsならSHA Pinning、DockerfileならSHA256digestでのPinningが可能です。  
同一バージョン番号で中身が変わっているタイプの侵害への対策として、有効にしている方も多いのではないでしょうか？  
aquaproj/aqua/aqua(以下、aqua)でもcommit SHAまたはchecksumを用いてPinningが可能です。
以下、検証の記録です。

## commit SHAでのPinning

hashicorp/terraform-config-inspect はGitHub Releasesを使用していないため、commit SHAをversionとして指定しています。

```yaml
---
registries:
  - type: standard
    ref: v4.520.2 # renovate: depName=aquaproj/aqua-registry
packages:
  - name: hashicorp/terraform-config-inspect
    # renovate: datasource=github-refs depName=hashicorp/terraform-config-inspect versioning=git ref=master
    version: "e06743db9cd8a4a96040fafd00503b6a9d357ff1"
```

問題点としては、renovateでのバージョンアップのために一工夫必要です。  
上記のようにrenovate用のコメントを追加する必要があります。  
terraform-config-inspectであれば機能追加がないため、ある程度バージョンアップしなくても問題ありません。  
しかし、全てのツールで同様に実施するのは煩雑ですので、checksumを利用してみます。

## aqua checksum

[開発者のsuzuki-shunsukeさんのzenn記事](https://zenn.dev/shunsuke_suzuki/articles/aqua-checksum-verification)
<https://aquaproj.github.io/docs/reference/config/checksum/>  
aqua.yamlにchecksumを設定します。

```yaml
---
checksum:
  enabled: true # By default, this is false
  require_checksum: true # By default, this is false
  supported_envs: # By default, all envs are supported
    - darwin
    - linux
registries:
  - #...
packages:
  - #...
```

require_checksumをtrueにすることで、checksumが設定されていないパッケージのインストールを禁止します。
`aqua update-checksum`すると、aqua-checksums.jsonが出力されます。  
以下はaquasecurity/trivyのchecksumです。

```json
{
  "checksums": [
    {
      "id": "github_release/github.com/aquasecurity/trivy/v0.70.0/trivy_0.70.0_Linux-64bit.tar.gz",
      "checksum": "8B4376D5D6BEFE5C24D503F10FF136D9E0C49F9127A4279FD110B727929A5AA9",
      "algorithm": "sha256"
    },
    {
      "id": "github_release/github.com/aquasecurity/trivy/v0.70.0/trivy_0.70.0_Linux-ARM64.tar.gz",
      "checksum": "2F6BB988B553A1BBAC6BDD1CE890F5E412439564E17522B88A4541B4F364FC8D",
      "algorithm": "sha256"
    },
    {
      "id": "github_release/github.com/aquasecurity/trivy/v0.70.0/trivy_0.70.0_macOS-64bit.tar.gz",
      "checksum": "52D531452B19E7593DA29366007D02A810E1E0080D02F9CF6A1AFB46C35AAA93",
      "algorithm": "sha256"
    },
    {
      "id": "github_release/github.com/aquasecurity/trivy/v0.70.0/trivy_0.70.0_macOS-ARM64.tar.gz",
      "checksum": "68E543C51DCC96E1C344053A4FDE9660CF602C25565D9F09DC17DD41E13B838A",
      "algorithm": "sha256"
    }
  ]
} 
```

このままだと、renovateからaqua.yamlが更新されるとCIが落ちてしまうので、checksumの更新も自動化します。  
<https://aquaproj.github.io/docs/guides/checksum>
公式ドキュメントではupdate-checksum-actionが利用されていますが、あえて手書きしてみます。  
(ドキュメントでも`Please consider autofix.ci or Securefix Action`との記載があることもあります)  
<https://github.com/aquaproj/update-checksum-workflow/blob/main/.github/workflows/update-checksum.yaml> を参考にしました。

```yaml
---
name: update-aqua-checksum
on:
  pull_request:
    paths:
      - aqua.yaml
      - aqua-checksums.json
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  update-aqua-checksums:
    runs-on: ubuntu-latest
    # checksum更新のためにpush権限を使うWorkflowなので、同一リポジトリ内のブランチからのPRに限定
    if: github.event.pull_request.head.repo.full_name == github.repository
    permissions:
      contents: write
      pull-requests: write
    steps:
      - name: Checkout
        timeout-minutes: 3
        uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      - uses: aquaproj/aqua-installer@11dd79b4e498d471a9385aa9fb7f62bb5f52a73c # v4.0.4
        timeout-minutes: 3
        with:
          aqua_version: v2.59.1
      - name: Update aqua-checksums.json
        timeout-minutes: 15
        run: aqua update-checksum -prune
      - name: Commit and push
        timeout-minutes: 3
        run: |
          set -eu
          if git diff --quiet aqua-checksums.json; then
            echo "aqua-checksums.json is up to date"
            exit 0
          fi
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add aqua-checksums.json
          git commit -m "chore(aqua): update aqua-checksums.json"
          git push
```

`-prune` を付けることで、現在の `aqua.yaml` で不要になったchecksumエントリも削除できます。
Renovateでバージョンが上がった後に古いchecksumを残したくないため、ここでは `-prune` を指定しています。

## まとめ

必須の対応ではありませんが、安心して利用するための仕組みとして用意されているので、試してみてはいかがでしょうか？  
