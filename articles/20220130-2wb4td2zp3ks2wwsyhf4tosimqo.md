---
title: "direnv + brew-php-switcherでPHPのバージョンを切り替える"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [php]
published: true
---

## What'S?

brewで複数のPHPバージョンをインストールして使っています。
phpenvも試したのですが依存関係を見つけてはinstallする、というのに疲れて続きませんでした...。
PythonやJavaScriptのようにPHPもプロジェクトごとにバージョン管理できればいいのになぁ...。
バージョンの切り替えが億劫なので、自動化できないか悩んでいたところ、数年ぶりに[PHP: the Right Way](http://ja.phptherightway.com/)を読んで、[brew-php-switcher](https://github.com/philcook/brew-php-switcher)が紹介されており、これでイケそうなのでdirenvと組み合わせてみました。

## requirements

- Mac
- brew
- direnv
- brew-php-switcher

## .envrc

早速、.envrcに以下を書き込みます。
使用しているPHPのバージョンが8.0と8.1のみなので↓こんなです。
必要に応じてgrepのパターン部分を修正ください。
関数化してdirenvrcに書き、.envrcでは呼び出すだけにするとさらにスッキリしますが、自分の環境ではまだそこまでは不要そうなので愚直に各.envrcに書きます。

```shell
if type brew-php-switcher &>/dev/null; then
  actual_php_version=$(/usr/local/bin/php --version \
                       | grep -m 1 -o "PHP 8.[0-9]" \
                       | cut -d' ' -f2)
  expect_php_version=$(jq -r .require.php <composer.json \
                       | cut -d'|' -f1 \
                       | grep -o "8\.[0-9]")
  [[ "$actual_php_version" != "$expect_php_version" ]] && brew-php-switcher "$expect_php_version" -s
fi
```

## 動作検証

まずはPHP 8.0を使用しているプロジェクトで試してみます。
PHP 8.1が有効な状態で、`direnv allow` してみます。

```shell
jq -r .require.php <composer.json | grep -o "8.[0-9]"
8.0
```

```shell
❯ php -v
PHP 8.1.2 (cli) (built: Jan 21 2022 04:47:46) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.1.2, Copyright (c) Zend Technologies
    with Zend OPcache v8.1.2, Copyright (c), by Zend Technologies
```

```shell
direnv allow
direnv: loading ~/src/github.com/watarukura/php-sample-project/.envrc
direnv: ([/usr/local/bin/direnv export fish]) is taking a while to execute. Use CTRL-C to give up.
Switching to php@8.0
Switching your shell
Unlinking /usr/local/Cellar/php@8.0/8.0.15.reinstall... 0 symlinks removed.
Unlinking /usr/local/Cellar/php/8.1.2.reinstall... 24 symlinks removed.
Linking /usr/local/Cellar/php@8.0/8.0.15.reinstall... 25 symlinks created.

If you need to have this software first in your PATH instead consider running:
  echo 'fish_add_path /usr/local/opt/php@8.0/bin' >> ~/.config/fish/config.fish
  echo 'fish_add_path /usr/local/opt/php@8.0/sbin' >> ~/.config/fish/config.fish
All done!
direnv: export ~XPC_SERVICE_NAME
```

もう一度実行してみます。

```shell
❯ direnv allow
direnv: loading ~/src/github.com/watarukura/php-sample-project/.envrc
direnv: export ~XPC_SERVICE_NAME
```

何も起こりません。成功のようです。

## 参考

手動でPHP 8.0と8.1を切り替えるfishスクリプトを書いて使っていたのですが、自動化できて不要になりました。
これで、来る8.2以降への切り替えの際も困らずに済みそうです。
供養においておきます。

```shell
function php80
  if contains "8.1" (/usr/local/bin/php --version | /usr/local/bin/ggrep -m 1 -oP "PHP 8.[0-9]" | cut -d' ' -f2)
    brew unlink php && brew link --force --overwrite php@8.0
  else
    echo "already php8.0"
  end
end

function php81
  if contains "8.0" (/usr/local/bin/php --version | /usr/local/bin/ggrep -m 1 -oP "PHP 8.[0-9]" | cut -d' ' -f2)
    brew unlink php@8.0 && brew link --force --overwrite php
  else
    echo "already php8.1"
  end
end
```
