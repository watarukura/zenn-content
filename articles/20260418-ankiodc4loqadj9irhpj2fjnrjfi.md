---
title: "自分専用のwikiを作る"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wiki"]
published: true
---

ローカルで動いて書き心地がcosenseのようなWikiが欲しい、とずっと探していました。  
既に社内用のドキュメントツールがある状況でcosenseの導入は難しいです。  
また、AI agentに読ませるならローカルのmarkdownファイルに書き出したい。  
Obsidianも使ってみましたが、IDE・ブラウザ・Slack・Ghosttyがデスクトップにひしめく中、1ウィンドウを占有するのも煩わしくて使わなくなってしまいました。  
ということで、自分専用のWikiを作ることにしました。**l**ocal-**w**iki.**m**d、略してlwmです。

<https://github.com/watarukura/lwm>

## 仕様

- ローカルで動作
- ファイルベース
- ブラウザから編集可能
    - 編集画面はvimキーバインドのみ利用可
        - yank時はclipboardにコピー
- 編集履歴はGitで管理
- ダークモード対応
- レスポンシブデザイン対応
- code blockのsyntax highlight対応
- 文字列検索機能
    - grepを実行
- 2 hop link、backlinkを表示
- 存在しないローカルlinkを踏むとページを新規作成する
- 1実行ファイルで動作

## やらないことリスト

- 認証機能を作らない
- vimキーバインドの有効・無効のトグルを作らない
- 全文検索・ベクトル検索のためにDBを導入しない
- 記事ページの階層化をしない

## 初期構築時の失敗

ChatGPTに相談して仕様を固めて、Junie CLI + Claude 4.6 Sonnetに実装してもらいました。

```markdown
ローカルで動かせるscrapbox.ioクローンを作りたいと思います。
scarpbox記法ではなくMarkdown記法でWYSIWYGで書けて、ローカルにあるmarkdownファイルを読み込めるようにしたい。
scrapbox.ioの特徴である2 hop link, back linkも実現したいです。
```

<!-- textlint-disable -->
::: details ChatGPTの回答
<!-- textlint-enable -->

```markdown
できます。実装方針としては、**「ローカル Markdown ファイルをソース・オブ・トゥルースにするデスクトップアプリ」**として作るのが一番きれいです。

私なら、まずは次の構成にします。

推奨構成

アプリ本体: Tauri
フロント: React または Svelte
エディタ: Tiptap か Milkdown
保存形式: .md
索引: SQLite
リンク解析: 独自の [[Wiki Link]] ルール

Tauri はローカルファイル操作のための file system plugin を持っていて、デスクトップアプリとして配布しやすいです。
Electron でも可能ですが、Electron は Chromium + Node.js を同梱する形です。
軽さを優先するなら、今回の用途は Tauri がかなり相性がいいです。

エディタは、要件を見ると Tiptap が第一候補です。
理由は、現在の Tiptap には Markdown の公式サポートがあり、Markdown を parse / serialize できます。
初期コンテンツを Markdown として読み込み、編集後に Markdown に戻す流れが取りやすいです。
Milkdown も「WYSIWYG Markdown editor」を明確に掲げていて有力ですが、拡張の自由度と周辺実装のしやすさでは Tiptap のほうが組み立てやすいです。
```

:::

というところから実際にTauriで動くアプリを作ってもらったのですが、大分過剰な実装でした。  
Junieのクレジット1ヶ月分があっという間に使いつくされました。  
動作はするものの、markdownファイルと全文検索のためのSQLiteの同期時にSQLiteが破損するなどのエラーが頻発。
自分では把握しきれないコードの山に加え、バグがたっぷりで実用に耐えないため、没にしてやり直します。

## 2回目のトライ

クレジットが回復するまでの間に設計を練り直します。  
初回と同様の設計・実装の分担ですが、仕様を削って最小限にします。

```markdown
markdownで書けるローカルで動作するwikiを作りたくて、.mdファイルを作成・更新できるものが欲しい。
認証不要、検索不要で、scrapboxの2 hoplinkとbacklinkが欲しい。
wikilinkではなくmarkdown linkで良いです。
```

<!-- textlint-disable -->
::: details ChatGPTの回答
<!-- textlint-enable -->

```markdown
その条件なら、既存Wikiを探すより自作の小さいローカルWikiの方が合っています。

前提として、Scrapboxの2-hop linkは、現在ページからリンクしている先と、その先を共有している関連ページを出す考え方です。
Obsidian界隈でも同等機能を求める文脈で説明されています。

結論としては、こういう構成が最も素直です。

- .md が実ファイル
- ローカルで HTTP サーバーを起動
- ブラウザで閲覧・編集
- 保存はサーバー側でファイルに書き戻す
- 起動時または保存時にリンクグラフを再計算
- 各ページで
    - backlinks
    - 2-hop links
    - ページ一覧

認証不要・検索不要なら、機能はかなり絞れます。必要なのは実質これだけです。

最小機能
1. pages/*.md を読む
2. [Page Name](link) 形式の内部リンクを抽出する
3. ページ表示時に本文を描画する
4. 編集して保存すると .md を更新する
5. backlinks を出す
6. 2-hop links を出す
```

:::

cosense(scrapboxは旧称)のcloneを作りたい、だとcosenseの特徴となる機能をアレコレ作ろうとしてしまい、失敗しました。  
欲しいものをもっと小さく表現します。  
動くことを確認しながら、ビジネスロジックが1ファイルで収まるサイズに抑えつつ、step by stepで開発を進めていきます。  

開発当初はNode.js + honoでサーバを構築しましたが、1バイナリで動かしたくなったので、frontendのコードをGoのバイナリに埋め込んでみました。  
[k1LoW/mo: mo is a Markdown viewer that opens .md files in a browser.](https://github.com/k1LoW/mo)を参考にさせていただきました。  
いちいち所定のディレクトリへ移動してコマンド実行、というのが面倒だったので、記事の書き出し先ディレクトリを環境変数で扱えるようにして、`lwm`コマンドを実行したらWikiサーバが起動するようにします。  
コマンドは短いほうがいいので、`local-md-wiki`という野暮ったい名前で開発してきたのですが`lwm`にリポジトリ名を変更しました。

## 現時点の動作

閲覧画面: i, aキーを押すと編集画面に遷移します。

![img.png](https://github.com/watarukura/zenn-content/blob/main/articles/images/20260418_wiki_view.png?raw=true)

編集画面: CodeMirror-vim を使用しており、`:`を打つとコマンドも実行できます。  
`:w`で保存、`:q`で保存せず閲覧画面に遷移します。

![img.png](https://github.com/watarukura/zenn-content/blob/main/articles/images/20260418_wiki_edit.png?raw=true)

検索機能: `grep -nri`した結果を表示しているだけです。

![img.png](https://github.com/watarukura/zenn-content/blob/main/articles/images/20260418_wiki_search.png?raw=true)

## まとめ

自分の欲望だけを考え、自分で使うツールを作るのはとても楽しいですね！  
Vibe Codingらしい欲望駆動開発です。  
<!-- textlint-disable -->
あれも欲しいこれも欲しいと機能を追加するのに遠慮は要らないのですが、アプリケーションのサイズは小さく抑えたい。  
<!-- textlint-enable -->
(いざというときに気合で読める程度の量と複雑性でないと使い続けられないので...)  
自分のマシンでさえ動けばいいので、`git`や`grep`のようなコマンドが未installであることを心配しなくて良い。  
もうしばらく、欲望の赴くままに破綻なく成長させられるか、継続していじっていこうと思います。

まかり間違って使ってみたいという方は、現状はセキュリティよりも個人の利便性を優先しています。
そのため、あくまでローカル専用としての利用を前提としていますのでご承知おきください。
(READMEにも頭のところに書いています)
