---
title: "Rustを写経する環境を作る"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "juypter-notebook", "VS Code"]
published: true
---

## What's?

[詳解Rustプログラミング](https://www.shoeisha.co.jp/book/detail/9784798160221)を写経しています。
<!-- textlint-disable -->
最初はIntelliJ IDEAで書いていたのですが、補完があまり効かずVS Codeに乗り換えました。
<!-- textlint-enable -->
(IntelliJ IDEAの設定のどこでうまくいっていないのかまでは調べられてないです...)
ついでにとアレコレくっつけていったらゴツゴツしてきたのですが、なかなか快適なので紹介。
写経中のリポジトリはこちら。
[https://github.com/watarukura/rust_in_action_study](https://github.com/watarukura/rust_in_action_study)

## Required

- VS Code
- Docker

## devcontainer

VS Codeのdevcontainerを使います。
moldを使ってみたかったので[Faster Rust Incremental Builds in Docker](https://purton.tech/blog/faster-rust-incremental-builds/)を参考にしつつ、最新のv1.1を使えるように書き換えています。
<! textlint-disable -->
(あんまりmoldの恩恵を受けられているわけではないのですが、まぁ趣味プロジェクトなので...)
<! textlint-enable -->
fish shellユーザーなので、bashユーザの方はこの辺消しちゃってください。
ghコマンドも写経中にGitHub開くのが面倒で使ってます。ホント便利です。
`gh pr create` -> `gh pr check --watch` -> `gh pr merge` で良いので。

```dockerfile
FROM rust:1.59.0-slim-bullseye

# install dependencies for build mold 
RUN apt-get update && \
    TZ=Asia/Tokyo apt-get install -y tzdata && \
    apt-get install -y \
    build-essential \
    git \
    clang \
    lld \
    cmake \
    libstdc++-10-dev \
    libssl-dev \
    libxxhash-dev \
    zlib1g-dev \
    pkg-config

# install mold
ENV mold_version=v1.1
RUN git clone --branch "$mold_version" --depth 1 https://github.com/rui314/mold.git && \
    cd mold && \
    make -j$(nproc) CXX=clang++ && \
    make install && \
    mv /mold/mold /usr/bin/mold && \
    mv /mold/mold-wrapper.so /usr/bin/mold-wrapper.so && \
    make clean

# install development utilities
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
      | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
      | tee /etc/apt/sources.list.d/github-cli.list > /dev/null && \
    echo 'deb http://download.opensuse.org/repositories/shells:/fish:/release:/3/Debian_11/ /' \
      | tee /etc/apt/sources.list.d/shells:fish:release:3.list && \
    curl -fsSL https://download.opensuse.org/repositories/shells:fish:release:3/Debian_11/Release.key \
      | gpg --dearmor \
      | tee /etc/apt/trusted.gpg.d/shells_fish_release_3.gpg > /dev/null && \
    apt-get update && \
    apt-get install -y \
      gh \
      fish \
      vim \
      curl \
      libzmq3-dev \
      jupyter-notebook && \
    rm -rf /var/lib/apt/lists/*

# use from rust-analyzer
RUN rustup component add rust-src && \
    rustup component add rust-analysis && \
    rustup component add rls && \
    rustup component add rustfmt

# vscode
RUN groupadd -g 1000 vscode && \
    useradd -m -s /bin/bash -u 1000 -g 1000 vscode
USER vscode
WORKDIR /vscode
RUN mkdir -p /vscode/target && chown vscode:vscode /vscode/target

# development utilities
RUN mold -run cargo install cargo-edit && \
    mold -run cargo install evcxr_jupyter --no-default-features && \
    evcxr_jupyter --install
```

docker-compose.yml↓

```yaml
version: '3.4'
services:
  development:
    build: 
      context: .
      dockerfile: Dockerfile
    volumes:
      - ..:/vscode:delegated
      # For building dependencies always use a volume, you get waaaay better performance.
      - target:/vscode/target
    # Overrides default command so things don't shut down after the process ends.
    command: sleep infinity
    working_dir: /vscode
volumes:
  # The volume for cargo build
  target:
```

devcontainer.jsonはこちら。
rust-analyzerとjupyterのextensionを入れることで、devcontainerを起動するだけで準備ができて便利。

```json
{
  "name": "Rust",
  "dockerComposeFile": [
    "docker-compose.yml"
  ],
  "service": "development",
  "workspaceFolder": "/vscode",
  "runArgs": [
    "--cap-add=SYS_PTRACE",
    "--security-opt",
    "seccomp=unconfined"
  ],
  "settings": {
    "lldb.executable": "/usr/bin/lldb",
    // VS Code don't watch files under ./target
    "files.watcherExclude": {
      "**/target/**": true
    }
  },
  "extensions": [
    "matklad.rust-analyzer",
    "bungcip.better-toml",
    "vadimcn.vscode-lldb",
    "mutantdino.resourcemonitor",
    "ms-toolsai.jupyter"
  ],
  "remoteUser": "vscode"
}
```

## VS Code

.vscode/settings.jsonはこちら。
formatOnSaveがうまく動かなかったので、save時にcargo fmtを走らせるようにしています。

```json
{
    "workbench.colorCustomizations": {
        // Name of the theme you are currently using
        "[Default Dark+]": {
            "rust_analyzer.inlayHints.foreground": "#868686f0",
            "rust_analyzer.inlayHints.background": "#3d3d3d48",
            // Overrides for specific kinds of inlay hints
            "rust_analyzer.inlayHints.foreground.typeHints": "#fdb6fdf0",
            // "rust_analyzer.inlayHints.foreground.paramHints": "#fdb6fdf0",
            "rust_analyzer.inlayHints.background.chainingHints": "#6b0c0c81"
        }
    },
    "editor.semanticTokenColorCustomizations": {
        "rules": {
            "*.mutable": {
                "fontStyle": "", // underline is the default
            },
            "operator.unsafe": "#ff6600",
            "function.unsafe": "#ff6600",
            "method.unsafe": "#ff6600"
        }
    },
    "rust-analyzer.checkOnSave.command": "fmt",
    "files.autoSave": "afterDelay",
    // "editor.formatOnSave": true,
    "editor.formatOnPaste": true,
    "terminal.integrated.profiles.linux": {
        "fish": {
            "path": "/usr/bin/fish"
        }
    },
    "terminal.integrated.defaultProfile.linux": "fish"
}
```

## Jupyter Notebook

当初はJupyterLabを別で起動していたのですが、VS Codeのjupyter拡張つかったらとても便利で鞍替え。
補完も効きますし、vscodevimを入れておけばコードはvimライクに書けます。
[evcxr](https://github.com/google/evcxr)をinstallしてRustが動くようにしました。
小さめのコードはJupyter、大きめのコードはVS Codeで写経しています。
fn main() はJuypterでは動かないみたいですね。(Jupyter弱者なので試行錯誤中...)
コード片貼り付けてどう動いているか見る、みたいなのにすごく便利です。

## GitHub Actions

lintとformatチェックだけ動かしています。
testは、写経の中でてきたら追加しようかなと。

```yaml
name: Lint and Test

on:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Setup code
        uses: actions/checkout@v2

      - name: Setup Rust toolchain
        uses: actions-rs/toolchain@v1
        with:
          profile: minimal
          toolchain: stable
          override: true
          components: rustfmt, clippy

      # https://github.com/actions/cache/blob/master/examples.md#rust---cargo
      - name: Cache cargo files
        uses: actions/cache@v2
        with:
          path: |
            ~/.cargo/bin/
            ~/.cargo/registry/index/
            ~/.cargo/registry/cache/
            ~/.cargo/git/db/
            target/
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

      - name: delete mold for CI
        run: |
          rm -f .cargo/config.toml

      - name: Check format
        uses: actions-rs/cargo@v1
        with:
          command: fmt
          args: --all -- --check

      - name: Run lint
        uses: actions-rs/cargo@v1
        with:
          command: clippy
          args: --all-features -- -D warnings
```

## rust-analyzer

複数workspaceにするとrust-analyzerがうまく動かないみたいです。
ですので、コード補完したい場合はディレクトリ構造の見直しが必要...。
参考: [[rust-analyzer]VScodeでRustの補完機能を使いたいのに動かない](https://zenn.dev/fah_72946_engr/articles/cf53487d3cc5fc)

## まとめ

詳解Rustプログラミングは残り半分くらいなのですが、写経継続していこうと思います。
Rustにこれから入門する、という方にもお勧めです。定番のcrate(rand、clap、serdeなど)についても必要になってから使い方を教えてくれます。
また、システムプログラミングの学習にも好適です。
自分はいまだにポインタなどが理解できているとは言いにくいのですが、読む前よりは多少理解が進んだかなと...。

写経環境を改善してたらヤクの毛刈りが楽しくなってアレコレやりました、という記事でした。
