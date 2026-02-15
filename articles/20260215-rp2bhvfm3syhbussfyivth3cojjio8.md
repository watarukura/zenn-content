---
title: "PHP7.2環境を構築する"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["php", "nix"]
published: true
---

諸事情あって、Apple SiliconのmacOS上にPHP 7.2環境が必要になったため、構築します。  
homebrewでは、PHPは8.1以降しかないので、NG。  
miseとnixを試してみます。  
どちらも駄目なら、諦めてdocker or ソースコードからbuildですね。

## mise

2026年2月現在、PHP 8.0以降であれば特段問題なくinstallできます。  
事前に関連ライブラリをbrewでinstallの上、mise.toml に↓のように書けばOK。
[参考: asdf](https://github.com/asdf-community/asdf-php/blob/248e9c6e2a7824510788f05e8cee848a62200b65/.github/workflows/workflow.yml#L52)

```shell
❯ brew install autoconf automake bison freetype gd gettext icu4c krb5 libedit libiconv libjpeg libpng libxml2 libzip pkg-config re2c zlib
❯ mise install php 8.4
❯ mise use php@8.4
php@8.4.18                                                                                                    ◜
mise ~/.local/share/chezmoi/mise.toml tools: php@8.4.18
❯ php -v
PHP 8.4.18 (cli) (built: Feb 15 2026 12:08:54) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.4.18, Copyright (c) Zend Technologies
```

さて、PHP 7.2だとどうなるかと言うと..。

```shell
❯ mise install php 7.2
#...(前半略)
checking for pkg-config... /opt/homebrew/bin/pkg-config
checking for OpenSSL version... >= 1.0.1
checking for CRYPTO_free in -lcrypto... yes
checking for SSL_CTX_set_ssl_version in -lssl... yes
checking for PCRE library to use... bundled
checking whether to enable PCRE JIT functionality... no
checking whether to enable the SQLite3 extension... yes
checking bundled sqlite3 library... yes
checking for ZLIB support... yes
checking if the location of ZLIB install directory is defined... no
configure: error: Cannot find zlib
php@7.2.34      checking if the location of ZLIB install directory is defined... no                           ◞
mise ERROR Failed to install asdf:php@7.2: ~/.local/share/mise/plugins/php/bin/install exited with non-zero status: exit code 1
mise ERROR Run with --verbose or MISE_VERBOSE=1 for more information
```

miseはasdf-phpを使用するので、以下を参考にPKG_CONFIGURE_OPTIONSを宣言しながら実行するラッパーを書きます。
<https://github.com/jdx/mise/discussions/4720#discussioncomment-14808686>
↓以下は最終型ですが、だいぶ紆余曲折ありました...。

<!-- markdownlint-disable MD013 -->
```shell
❯ cat install_php.bash
#!/usr/bin/env bash
set -euo pipefail

# Homebrew prefixes
LIBZ_PREFIX=$(brew --prefix zlib)
LIBPNG_PREFIX=$(brew --prefix libpng)
JPEG_PREFIX=$(brew --prefix jpeg)
WEBP_PREFIX=$(brew --prefix webp)
FREETYPE_PREFIX=$(brew --prefix freetype)
LIBXPM_PREFIX=$(brew --prefix libxpm)
LIBX11_PREFIX=$(brew --prefix libx11)
BZIP2_PREFIX=$(brew --prefix bzip2)
CURL_PREFIX=$(brew --prefix curl)
GETTEXT_PREFIX=$(brew --prefix gettext)
GMP_PREFIX=$(brew --prefix gmp)
MHASH_PREFIX=$(brew --prefix mhash)
OPENSSL_PREFIX=$(brew --prefix openssl@1.1)
READLINE_PREFIX=$(brew --prefix readline)
LIBSODIUM_PREFIX=$(brew --prefix libsodium)
LIBXSLT_PREFIX=$(brew --prefix libxslt)
BINUTILS_PREFIX=$(brew --prefix binutils)
LIBICONV_PREFIX=$(brew --prefix libiconv)
SQLITE_PREFIX=$(brew --prefix sqlite)
XORGPROTO_PREFIX=$(brew --prefix xorgproto)
ICU4C_PREFIX=$(brew --prefix icu4c)

export PHP_CONFIGURE_OPTIONS="--with-pear \
  --without-pcre-jit \
  --enable-fpm \
  --with-fpm-user=$(whoami) \
  --with-fpm-group=staff \
  --enable-bcmath \
  --enable-calendar  \
  --enable-dba  \
  --enable-exif  \
  --enable-ftp  \
  --enable-intl  \
  --enable-mbstring  \
  --enable-mysqlnd \
  --enable-pcntl \
  --enable-shmop \
  --enable-soap \
  --enable-sockets \
  --enable-sysvmsg \
  --enable-sysvsem \
  --enable-sysvshm \
  --enable-zip \
  --with-gd \
  --with-bz2=${BZIP2_PREFIX} \
  --with-curl=${CURL_PREFIX} \
  --with-gettext=${GETTEXT_PREFIX} \
  --with-gmp=${GMP_PREFIX} \
  --with-freetype-dir=${FREETYPE_PREFIX} \
  --with-jpeg-dir=${JPEG_PREFIX} \
  --with-png-dir=${LIBPNG_PREFIX} \
  --with-webp-dir=${WEBP_PREFIX} \
  --with-xpm-dir=${LIBXPM_PREFIX} \
  --with-mhash=${MHASH_PREFIX} \
  --with-mysqli=mysqlnd \
  --with-openssl=${OPENSSL_PREFIX} \
  --with-pdo-mysql=mysqlnd \
  --with-readline=${READLINE_PREFIX} \
  --with-sodium=${LIBSODIUM_PREFIX} \
  --with-iconv=${LIBICONV_PREFIX} \
  --with-xsl=${LIBXSLT_PREFIX} \
  --with-sqlite3=${SQLITE_PREFIX} \
  --with-pdo-sqlite=${SQLITE_PREFIX} \
  --with-zlib=${LIBZ_PREFIX} \
  --with-zlib-dir=${LIBZ_PREFIX}"

export LDFLAGS="-L${LIBICONV_PREFIX}/lib -L${READLINE_PREFIX}/lib -L${SQLITE_PREFIX}/lib -L/opt/homebrew/lib -L${OPENSSL_PREFIX}/lib -L${LIBZ_PREFIX}/lib -L${LIBPNG_PREFIX}/lib -L${JPEG_PREFIX}/lib -L${WEBP_PREFIX}/lib -L${FREETYPE_PREFIX}/lib -L${LIBXPM_PREFIX}/lib -L${LIBX11_PREFIX}/lib"
export CPPFLAGS="-I${LIBICONV_PREFIX}/include -I${READLINE_PREFIX}/include -I${SQLITE_PREFIX}/include -I/opt/homebrew/include -I${OPENSSL_PREFIX}/include -I${LIBZ_PREFIX}/include -I${LIBPNG_PREFIX}/include -I${JPEG_PREFIX}/include -I${WEBP_PREFIX}/include -I${FREETYPE_PREFIX}/include/freetype2 -I${LIBXPM_PREFIX}/include -I${LIBX11_PREFIX}/include -I${XORGPROTO_PREFIX}/include -DU_DEFINE_FALSE_AND_TRUE=1 -DU_USING_ICU_NAMESPACE=1"
export LIBS="${LIBICONV_PREFIX}/lib/libiconv.dylib ${READLINE_PREFIX}/lib/libreadline.dylib ${SQLITE_PREFIX}/lib/libsqlite3.dylib -lncurses -lX11 -lz -lresolv"
export CXXFLAGS="-std=c++17"
export CFLAGS="-Wno-implicit-function-declaration -Wno-incompatible-function-pointer-types"
export PKG_CONFIG_PATH="${ICU4C_PREFIX}/lib/pkgconfig:${LIBZ_PREFIX}/lib/pkgconfig:${LIBPNG_PREFIX}/lib/pkgconfig:${JPEG_PREFIX}/lib/pkgconfig:${WEBP_PREFIX}/lib/pkgconfig:${FREETYPE_PREFIX}/lib/pkgconfig:${LIBXPM_PREFIX}/lib/pkgconfig:${LIBX11_PREFIX}/lib/pkgconfig:${OPENSSL_PREFIX}/lib/pkgconfig:${SQLITE_PREFIX}/lib/pkgconfig:${PKG_CONFIG_PATH:-}"
export PATH="${BINUTILS_PREFIX}/bin:$PATH"

# Plugin can't apply the patch, so we do it by wrapping 'pkg-config'
# which is called by 'configure' when we are already in the source directory.
ORIG_PKG_CONFIG=$(which pkg-config)
PATCH_PATH="${PWD}/php72_icu.patch"
cat <<EOF > pkg-config-wrapper
#!/usr/bin/env bash
ORIG_PKG_CONFIG="${ORIG_PKG_CONFIG}"
PATCH_PATH="${PATCH_PATH}"

if [[ "\$*" == *"--exists"* ]] && [[ "\$*" == *"icu-i18n"* ]]; then
    # We need to wait until the source is extracted and we are in the source directory.
    # PHP's configure script calls pkg-config several times.
    # We look for a file that exists in the extracted source to apply the patch.
    if [ -f ext/intl/breakiterator/codepointiterator_internal.h ]; then
        # When configure runs, it's already inside the php source directory.
        if ! grep -q "virtual bool operator==" ext/intl/breakiterator/codepointiterator_internal.h; then
            # Force use sed for reliability
            sed -i '' 's/virtual UBool operator==/virtual bool operator==/' ext/intl/breakiterator/codepointiterator_internal.h
            sed -i '' 's/UBool CodePointBreakIterator::operator==/bool CodePointBreakIterator::operator==/' ext/intl/breakiterator/codepointiterator_internal.cpp
            # Fix readdir_r detection/usage on macOS
            if [ -f main/reentrancy.c ]; then
                # Handle various readdir_r patterns
                sed -i '' 's/\*result = readdir_r(dirp, entry)/readdir_r(dirp, entry, result)/' main/reentrancy.c
                sed -i '' 's/readdir_r(dirp, entry)/readdir_r(dirp, entry, result)/' main/reentrancy.c
                echo "Applied readdir_r fix in current directory" >&2
            fi
            touch .patched_icu
            echo "Applied ICU fix via sed in current directory" >&2
        elif [ ! -f .patched_icu ]; then
            touch .patched_icu
            echo "ICU already patched in current directory (detected by content)" >&2
        fi
    else
        # If we are not in the source directory yet, maybe we can find it.
        # asdf-php extracts to a temporary directory.
        # Let's try to find where we are and if the file is reachable.
        TARGET_FILE=\$(find /var/folders -name codepointiterator_internal.h 2>/dev/null | grep "ext/intl/breakiterator/codepointiterator_internal.h" | head -n 1)
        if [ -n "\${TARGET_FILE:-}" ]; then
            BASE_DIR=\${TARGET_FILE%/ext/intl/breakiterator/codepointiterator_internal.h}
            echo "Found PHP source at \$BASE_DIR" >&2
            # Force re-patching attempt even if .patched_icu exists because it might have failed partially
            # or we need to ensure sed runs if patch was skipped.
            # We use a subshell to avoid changing the current directory of the wrapper if it matters, 
            # though pushd/popd should handle it.
            (
                cd "\$BASE_DIR"
                if [ ! -f .patched_icu ] || ! grep -q "virtual bool operator==" ext/intl/breakiterator/codepointiterator_internal.h; then
                    # Check if the file is already patched
                    if ! grep -q "virtual bool operator==" ext/intl/breakiterator/codepointiterator_internal.h; then
                        # Force use sed because patch seems unreliable or already partially applied
                        echo "Applying ICU fix via sed to ensure consistency" >&2
                        sed -i '' 's/virtual UBool operator==/virtual bool operator==/' ext/intl/breakiterator/codepointiterator_internal.h
                        sed -i '' 's/UBool CodePointBreakIterator::operator==/bool CodePointBreakIterator::operator==/' ext/intl/breakiterator/codepointiterator_internal.cpp
                        # Fix readdir_r detection/usage on macOS
                        if [ -f main/reentrancy.c ]; then
                            # Handle various readdir_r patterns
                            sed -i '' 's/\*result = readdir_r(dirp, entry)/readdir_r(dirp, entry, result)/' main/reentrancy.c
                            sed -i '' 's/readdir_r(dirp, entry)/readdir_r(dirp, entry, result)/' main/reentrancy.c
                            echo "Applied readdir_r fix in \$BASE_DIR" >&2
                        fi
                        touch .patched_icu
                        echo "Applied ICU fix via sed in \$BASE_DIR" >&2
                    else
                        touch .patched_icu
                        echo "ICU already patched in \$BASE_DIR" >&2
                    fi
                fi
            )
        fi
    fi
fi
exec "\${ORIG_PKG_CONFIG}" "\$@"
EOF
chmod +x pkg-config-wrapper

ASDF_PHP_PATCHURL="${PWD}/php72_icu.patch" \
PKG_CONFIG="${PWD}/pkg-config-wrapper" \
  mise install php@7.2

rm pkg-config-wrapper
```
<!-- markdownlint-enable -->

```shell
./install_php.bash >install_log 2>&1 || tail install_log
```

実行して、エラーが出ていたら、エラーログを読ませてLLMにデバッグしてもらいます。

```shell
❯ junie ' @install_log は @install_php.bash の実行ログです。エラーの原因を調べてください。どのような対応が必要 かわかったら、install_php.bashを修正してください。'
```

- PHPのbuildオプションにもバージョンに依る違いがある(--with-zip と --enable-zip など)
- brewでのライブラリのinstallが前提
- icu4c@68が必要だけどもう公開されてなくてicu4c@78にパッチを作った

サポート期限切れのバージョンを利用するのは、かくも難しい...。  
自分ひとりではとても完了しそうになかったですが、Gemini 3 Flash, GPT-5.2-codex, Claude 4.6 Opusに相談して、丸一日かかってようやくinstallできました。

```shell
❯ mise use php@7.2.34
php@7.2.34                                                                                                    ◜
mise ~/Documents/20260214_mise_php/mise.toml tools: php@7.2.34
❯ mise list php
Tool  Version  Source                                   Requested
php   7.2.34   ~/Documents/20260214_mise_php/mise.toml  7.2.34
php   8.3.30
php   8.4.18
❯ php -v
PHP 7.2.34 (cli) (built: Feb 15 2026 11:39:18) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
```

調べている中で、build済みのバイナリをダウンロードして使うstatic-phpというプロジェクトを見つけました。  
こちらも、PHP 7.2は対象外でしたが...。

<https://github.com/jdx/mise/discussions/4720>
<https://github.com/crazywhalecc/static-php-cli>

```shell
mise use ubi:adwinying/php@8.1 # or any version you like
```

## nix

方法は2つ。

1. <https://lazamar.co.uk/nix-versions/?channel=nixpkgs-unstable&package=php> でnixpkgsの古いバージョンを調べ、tarballをGitHubから取ってくる
2. <https://github.com/fossar/nix-phps> を使う

1.の方はすぐ試せるのですが、うまくいきませんでした。

```shell
❯ nix-shell -p php72base -I nixpkgs=https://github.com/NixOS/nixpkgs/archive/4f036a554865766d662da4b416e5390940eaed09.tar.gz
unpacking 'https://github.com/NixOS/nixpkgs/archive/4f036a554865766d662da4b416e5390940eaed09.tar.gz' into the Git cache...
error:
       … while calling the 'derivationStrict' builtin
         at «nix-internal»/derivation-internal.nix:37:12:
           36|
           37|   strict = derivationStrict drvAttrs;
             |            ^
           38|

       … while evaluating derivation 'shell'
         whose name attribute is located at /nix/store/y956dmrl0s5rmb28n2r6csykpa0dp59z-source/pkgs/build-support/trivial-builders.nix:7:15

       … while evaluating attribute 'buildInputs' of derivation 'shell'
         at /nix/store/y956dmrl0s5rmb28n2r6csykpa0dp59z-source/pkgs/stdenv/generic/make-derivation.nix:220:11:
          219|           depsHostHost                = lib.elemAt (lib.elemAt dependencies 1) 0;
          220|           buildInputs                 = lib.elemAt (lib.elemAt dependencies 1) 1;
             |           ^
          221|           depsTargetTarget            = lib.elemAt (lib.elemAt dependencies 2) 0;

       (stack trace truncated; use '--show-trace' to show the full, detailed trace)

       error: undefined variable 'php72base'
       at «string»:1:107:
            1| {...}@args: with import <nixpkgs> args; (pkgs.runCommandCC or pkgs.runCommand) "shell" { buildInputs = [ (php72base) ]; } ""
             |                                                                                                           ^
```

`--show-trace` した上でエラーログをCodexに渡したところ、以下が原因とのこと。

```text
このトレースは「php72base 自体は見つかったが、その依存（古い pcre や xz など）を fetch しようとした時に null が混ざっていて落ちている」ケースに見えます。
nixpkgs 単体で古い PHP をビルドするルートは、古いミラーや削除された URL に当たって壊れやすいです。
安定させるなら、やはり nix-phps を使う方が現実的です。nix-phps は古い PHP のビルド定義や依存の互換性を保つ目的でメンテされているので、php72 が生きています。
もしこの nix-shell -p php72base をどうしても使いたいなら、次のどちらかが必要です。

nixpkgs の特定リビジョン（依存の URL がまだ生きている時点）を探して固定する

fetchurl のミラーや pcre の URL をオーバーライドして復旧させる
どちらで進めたいですか？
今の flake.nix は nix-phps 前提で動くように整えてあるので、まずは nix develop を通したいならそちらで詰めていくのが早いです。
```

ということで、2.で試してみます。  
↓以下、flake.nixです。

```nix
{
  description = "clinic.docker dev shell with PHP 7.2";

  inputs = {
    nix-phps.url = "github:fossar/nix-phps";
    nixpkgs.follows = "nix-phps/nixpkgs";
  };

  outputs = {
    self,
    nixpkgs,
    nix-phps,
  }: let
    systems = [
      "x86_64-linux"
      "aarch64-darwin"
    ];
    forAllSystems = f: nixpkgs.lib.genAttrs systems (system: f system);
  in {
    devShells = forAllSystems (system: let
      pkgs = import nixpkgs {
        inherit system;
        overlays = [nix-phps.overlays.default];
      };
    in {
      default = pkgs.mkShell {
        packages = let
          php = pkgs.php72.withExtensions ({
            enabled,
            all,
          }:
            enabled ++ [all.redis all.xdebug]);
        in [
          php
          pkgs.php72.packages.composer
        ];
      };
    });
  };
}
```

```shell
❯ nix develop                                                                                                                            
evaluation warning: The xorg package set has been deprecated, 'xorg.libXpm' has been renamed to 'libxpm'
bash-5.3$ php -v
PHP 7.2.34 (cli) (built: Sep 30 2020 05:15:56) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
    with Zend OPcache v7.2.34, Copyright (c) 1999-2018, by Zend Technologies
    with Xdebug v3.1.6, Copyright (c) 2002-2022, by Derick Rethans
bash-5.3$ php -m
[PHP Modules]
bcmath
calendar
Core
ctype
curl
date
dom
exif
fileinfo
filter
ftp
gd
gettext
gmp
hash
iconv
intl
json
ldap
libxml
mbstring
mysqli
mysqlnd
openssl
pcntl
pcre
PDO
pdo_mysql
PDO_ODBC
pdo_pgsql
pdo_sqlite
pgsql
Phar
posix
readline
redis
Reflection
session
SimpleXML
soap
sockets
sodium
SPL
sqlite3
standard
sysvsem
tokenizer
xdebug
xml
xmlreader
xmlwriter
Zend OPcache
zip
zlib

[Zend Modules]
Xdebug
Zend OPcache

bash-5.3$ 
```

はい、利用できるようになりました！
RedisとXDebugもついでに入っています。

## docker

これが一番カンタンかもですね。

```shell
❯ docker run -it php:7.2 /bin/bash -c 'php -v'
Unable to find image 'php:7.2' locally
7.2: Pulling from library/php
7713373f413a: Pull complete 
f244ac071252: Pull complete 
a22d47705f6c: Pull complete 
c17a0a78e91d: Pull complete 
07b2c682d715: Pull complete 
3a7edbfad163: Pull complete 
c9648d7fcbb6: Pull complete 
f88cecc04e76: Pull complete 
30eb7a300f13: Pull complete 
7012ad2a4feb: Pull complete 
Digest: sha256:42ffbc0798e4449bbd1e14fc4dcb87774aa1ad1900a09ef6a965bc0880aa2161
Status: Downloaded newer image for php:7.2
PHP 7.2.34 (cli) (built: Dec 11 2020 11:14:11) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
```

## まとめ

サポート期限切れのバージョンを使うのはとても大変なのでバージョンアップは計画的に行いましょう。
