---
title: "postfixサーバをFargateに移植する"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws"]
published: true
---

副業先で、EC2で動いているpostfixサーバをFargate移植することになったので記録しておきます。  
リレーサーバとしては使わず、メールを受信けっしてログを書いたら廃棄するという役割。  
副業先の上司がポチポチっと数分で作ったものです。

```mermaid
graph LR
    client -->|SMTP| postfix
    subgraph AWS 
        subgraph EC2
            postfix
        end
    end
```

これをterraform化して、Fargateで動作させたい。

## Gitリポジトリを作る

とりあえずGitHubにリポジトリを作り、renovateを有効化します。  
初手、aquaをinstall。

```shell
brew install aquaproj/aqua/aqua
```

何はなくともterraformをinstallします。

```Shell
aqua g -i hashicorp/terraform
```

## CIを動かす準備

まず、OIDCを設定します。
アプリケーションやネットワークなどと関係ないものですので、settingsというtfstateを切ります。

```Shell
mkdir -p terraform/settings
cd terraform/settings
touch oidc.tf
```

OIDCの設定用にoidc.tfに書き、最初はローカルからplan / applyします。  
IAM role、provider等ができたところで、GitHub ActionsでCIを動かせるようにします。

```Shell
mkdir .github/workflows
cd .github/workflows
touch plan.yml
touch apply.yml
```

on.pull_requestでplanを実行、PRがマージされたらapplyします。
ついでに、plan結果をPRにコメントしてほしいので、tfcmtを入れます。
lintもしたいのでtflint、trivyを入れます。

```Shell
aqua g -i suzuki-shunsuke/tfcmt
aqua g -i terraform-linters/tflint
aqua g -i aquasecurity/trivy
touch lint.yml
```

lefthookでpre-commitフックでも実行するようにして、ローカル・リモートの双方でlintが走るようにします。

```Shell
aqua g -i evilmartians/lefthook
lefthook install
```

## terraform importする

tfstateをゆるく分割します。ネットワークとアプリケーションは分離します。  
ネットワークはそんなに更新されないので、分離したほうが安心。

```Shell
mkdir -p terraform/network
cd terraform/network
touch main.tf
touch vpc.tf
touch subnet.tf
touch import.tf
```

import.tfにはimportブロックを書き、差分がでないところまでresourceのプロパティを書きます。  
importが終わったらimport.tfは削除します。
ネットワークのあとは、EC2・Route53をimportします。

```Shell
mkdir -p terraform/app
cd terraform/app
touch main.tf
touch iam.tf
touch ec2.tf
touch route53.tf
touch import.tf
```

これで、既存のリソースのimportが終わりました。

## postfixをdockerで動かす

既存のpostfixからmain.cfをコピってきて、dockerで動くようにします。

```Shell
mkdir docker
touch Dockerfile
touch main.cf
touch entrypoint.sh
```

`docker run` で動かせるようになったら、ECRにpushできるようにします。

```Shell
mkdir -p terraform/app
cd terraform/app
touch ecr.tf
```

ECRができたら、CIへのpush用workflowを作ります。

```Shell
cd .github/workflows
touch deploy.yml
```

## NLB + ECSを作る

SMTPを通したい・AutoScalingしたいので、NLBをECSの前に置きます。

```Shell
cd terraform/app
touch nlb.tf
touch ecs.tf
```

できあがった構成は↓こんな感じ。

```mermaid
graph LR
    client -->|SMTP| NLB
    subgraph AWS
        NLB --> postfix
        subgraph Fargate
            postfix
        end
    end
```

## 切り替え

Route53の加重ルーティングで既存10%/新90%で振り分けします。
問題なく動作するようなら、50%/50% -> 0%/100%という順序にしました。  
無事に切り替えられたら、EC2を削除します。
