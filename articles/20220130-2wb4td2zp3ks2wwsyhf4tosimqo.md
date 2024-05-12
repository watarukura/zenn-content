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
PythonやJavaScriptのようにPHPもプロジェクトごとにバージョン管理できればよいのになぁ...。
数年ぶりに[PHP: the Right Way](http://ja.phptherightway.com/)を読んだところ、[brew-php-switcher](https://github.com/philcook/brew-php-switcher)が紹介されており、これでイケそうなのでdirenvと組み合わせてみました。

## requirements

- Mac
- brew
- direnv
- brew-php-switcher

現状、以下のようになっています。
バージョンの記載がないphpは、PHP 8.1です。

```shell
brew list | grep php
brew-php-switcher
php
php@8.0
```

## .envrc

さっそく、.envrcに以下を書き込みます。
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

PHP 8.1が有効な状態でPHP 8.0を使用しているプロジェクトを開いた際にバージョンが切り替わるか試してみます。

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

.envrcを修正して、`direnv allow`します。

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

```shell
php -v
PHP 8.0.15 (cli) (built: Jan 21 2022 04:49:41) ( NTS )
Copyright (c) The PHP Group
Zend Engine v4.0.15, Copyright (c) Zend Technologies
    with Zend OPcache v8.0.15, Copyright (c), by Zend Technologies
```

無事に切り替わりました！
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

## 2022/03/06 追記

<!-- textlint-disable -->
PECL installしたときにPECLやphpizeに`/usr/local/Cellar/php@8.0/8.0.16/bin/php is not found` って怒られるので、原因を調べてみました。
<!-- textlint-enable -->
brewで再インストールすると、バージョン番号.reinstallっていうディレクトリ名にインストールされる模様。

PECLとphpizeを書き換えてやると無事に動きました。
もっとよい方法がありそうですが..。

```shell
❯ which php
/usr/local/bin/php
❯ ls -l /usr/local/bin/php
lrwxr-xr-x 1 watarukura admin 42  3  2 05:58 /usr/local/bin/php -> ../Cellar/php@8.0/8.0.16.reinstall/bin/php
❯ which pecl
/usr/local/bin/pecl
❯ which phpize
/usr/local/bin/phpize
❯ sudo sed -i.org -e 's/8.0.16/8.0.16.reinstall/g' /usr/local/bin/pecl
❯ sudo sed -i.org -e 's/8.0.16/8.0.16.reinstall/g' /usr/local/bin/phpize
```

```diff
❯ diff -u /usr/local/bin/pecl.org /usr/local/bin/pecl
--- /usr/local/bin/pecl.org 2022-03-06 13:27:42.073462374 +0900
+++ /usr/local/bin/pecl 2022-03-02 15:57:12.987867522 +0900
@@ -4,10 +4,10 @@
 if test "x$PHP_PEAR_PHP_BIN" != "x"; then
   PHP="$PHP_PEAR_PHP_BIN"
 else
-  if test "/usr/local/Cellar/php@8.0/8.0.16/bin/php" = '@'php_bin'@'; then
+  if test "/usr/local/Cellar/php@8.0/8.0.16.reinstall/bin/php" = '@'php_bin'@'; then
     PHP=php
   else
-    PHP="/usr/local/Cellar/php@8.0/8.0.16/bin/php"
+    PHP="/usr/local/Cellar/php@8.0/8.0.16.reinstall/bin/php"
   fi
 fi

@@ -16,12 +16,12 @@
   INCDIR=$PHP_PEAR_INSTALL_DIR
   INCARG="-d include_path=$PHP_PEAR_INSTALL_DIR"
 else
-  if test "/usr/local/Cellar/php@8.0/8.0.16/share/php@8.0/pear" = '@'php_dir'@'; then
+  if test "/usr/local/Cellar/php@8.0/8.0.16.reinstall/share/php@8.0/pear" = '@'php_dir'@'; then
     INCDIR=`dirname $0`
     INCARG=""
   else
-    INCDIR="/usr/local/Cellar/php@8.0/8.0.16/share/php@8.0/pear"
-    INCARG="-d include_path=/usr/local/Cellar/php@8.0/8.0.16/share/php@8.0/pear"
+    INCDIR="/usr/local/Cellar/php@8.0/8.0.16.reinstall/share/php@8.0/pear"
+    INCARG="-d include_path=/usr/local/Cellar/php@8.0/8.0.16.reinstall/share/php@8.0/pear"
   fi
 fi
```

```diff
❯ diff -u /usr/local/bin/phpize.org /usr/local/bin/phpize
--- /usr/local/bin/phpize.org 2022-03-06 13:27:16.089971124 +0900
+++ /usr/local/bin/phpize 2022-03-02 15:57:49.274987158 +0900
@@ -1,8 +1,8 @@
 #!/bin/sh

 # Variable declaration
-prefix='/usr/local/Cellar/php@8.0/8.0.16'
-datarootdir='/usr/local/Cellar/php@8.0/8.0.16/share'
+prefix='/usr/local/Cellar/php@8.0/8.0.16.reinstall'
+datarootdir='/usr/local/Cellar/php@8.0/8.0.16.reinstall/share'
 exec_prefix="`eval echo ${prefix}`"
 phpdir="`eval echo ${exec_prefix}/lib/php`/build"
 includedir="`eval echo ${prefix}/include`/php"
@@ -152,7 +152,7 @@
 phpize_replace_prefix()
 {
   $SED \
-  -e "s#/usr/local/Cellar/php@8.0/8.0.16#$prefix#" \
+  -e "s#/usr/local/Cellar/php@8.0/8.0.16.reinstall#$prefix#" \
   < "$phpdir/phpize.m4" > configure.ac
 }
```

PECL、phpizeは両方ともシェルスクリプトなんですね。初めて知りました。

```shell
❯ pecl install --configureoptions 'enable-sockets="yes" enable-openssl="yes" enable-http2="yes" enable-mysqlnd="yes" enable-swoole-json="yes" enable-swoole-curl="yes"' openswoole-4.10.0
❯ php --ri openswoole

openswoole

Open Swoole => enabled
Author => Open Swoole Group <hello@openswoole.com>
Version => 4.10.0
Built => Mar  6 2022 13:08:25
coroutine => enabled with boost asm context
kqueue => enabled
rwlock => enabled
sockets => enabled
openssl => OpenSSL 1.1.1m  14 Dec 2021
dtls => enabled
http2 => enabled
json => enabled
curl-native => enabled
pcre => enabled
zlib => 1.2.11
brotli => E16777225/D16777225
mysqlnd => enabled
async_redis => enabled

Directive => Local Value => Master Value
swoole.enable_coroutine => On => On
swoole.enable_library => On => On
swoole.enable_preemptive_scheduler => Off => Off
swoole.display_errors => On => On
swoole.use_shortname => On => On
swoole.unixsock_buffer_size => 262144 => 262144
```

openswooleをinstallしてみました。無事にinstallできたようです。
