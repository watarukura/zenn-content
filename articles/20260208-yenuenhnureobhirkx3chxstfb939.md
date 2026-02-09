---
title: "todoコマンドを作る"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "cli"]
published: true
---

todoリストをCLIで管理したくなったので、todoコマンドを自作することにしました。  
できあがったものは↓こちらです。
<https://github.com/watarukura/todo>

## アイデアをまとめる

- `todo list`
    - todoの一覧を表示する
    - fzfで選択してtodoファイルを開く
- `todo cd`
    - todo管理ディレクトリに移動する
- `todo $title`
    - todoを追加する
    - `todo $title $project`
        - 複数プロジェクトを並行して進めることがあるため、目印としてprojectを設定できるようにする
- `todo done`
    - fzfで選択してfinished_atに当日の日付を入れ、doneディレクトリ配下に移動する
- todoは1件1ファイルとする
- メタデータはfront matterで登録する
    - title: タイトル
    - created_at: todo開始日
    - finished_at: todo終了日
    - project: プロジェクト
- todoファイル以外のデータストアは使用しない

## シェルスクリプトでプロトタイプを作る

[Front Matter | yq](https://mikefarah.gitbook.io/yq/usage/front-matter)  
yqコマンドでfront matterが読めるようなので、これを使うことにします。  
↓こんな感じでfront matterから取得したり、更新したりできます。

```shell
❯ cat <<EOF > sample.md
---
title: hoge
---
body
EOF

❯ yq --front-matter=extract '.title' sample.md
hoge

❯ yq --front-matter=process '. + {"created_at": now} ' sample.md
---
title: hoge
created_at: 2026-02-09T09:02:38+09:00
---
body
```

↓出来上がりはこれ。.zshrcに書き込みます。

```shell
function todo() {
  local todo_dir="$HOME/Documents/todo"
  local today=$(date +%Y-%m-%d)
  local escaped_filename=$(echo $1 | sed -e 's;/;_;g' -e 's;\x0;;g')

  if [[ $# -eq 1 ]]; then
    if [[ "$1" == "list" ]]; then
      ls "$todo_dir/"*.md |
        xargs basename |
        while read -r todo_file; do
          todo_file="$todo_file" \
          yq --front-matter=extract \
            '[.title, .created_at, .project|"", strenv(todo_file)] |@tsv' \
            "$todo_dir/$todo_file" |
            fzf | cut -f4 | xargs -I{} $EDITOR "$todo_dir/{}"
        done
      return
    fi
    if [[ "$1" == "cd" ]]; then
      cd $todo_dir
      return
    fi
    if [[ "$1" == "done" ]]; then
      donefile=$(ls "$todo_dir/"*.md | xargs -I{} basename {} | fzf)
      today="$today" \
        yq --front-matter=process '. + {"finished_at": strenv(today)}' "$todo_dir/$donefile" -i
      mkdir -p "$todo_dir/done/$today/"
      mv "$todo_dir/$donefile" "$todo_dir/done/$today/"
      return
    fi
    if [[ -e $todo_dir/$1.md ]]; then
      $EDITOR "$todo_dir/$1".md
      return
    fi

    printf -- "---\n{}\n---\n" >"$todo_dir/$escaped_filename.md"
    title="$1" today="$today" \
      yq --front-matter=process \
      '. + {"title": strenv(title), "created_at": strenv(today)}' \
      "$todo_dir/$escaped_filename.md" -i
    $EDITOR "$todo_dir/$escaped_filename".md
  elif [[ $# -eq 2 ]]; then
    printf -- "---\n{}\n---\n" >"$todo_dir/$escaped_filename.md"
    title="$1" today="$today" project="$2" \
      yq --front-matter=process \
      '. + {"title": strenv(title), "created_at": strenv(today), "project": strenv(project)}' \
      "$todo_dir/$escaped_filename.md" -i
    $EDITOR "$todo_dir/$escaped_filename".md
  else
    cd $todo_dir
    printf -- "---\n{}\n---\n" | $EDITOR
    cd -
  fi
}
```

これで、動くものはできました。

## AI Agentに読ませて別言語に翻訳する

シェルスクリプトだとテストが書きにくいですし、テストがないと、機能追加して壊れることがしょっちゅう発生します。  
(もちろん、書けないということはないですが...)  
ということで、Goに書き換えます。
[Junie-CLI](https://junie.jetbrains.com/)のpreviewに申し込んだところ、利用できるようになったので使ってみます。  

2026-02-08現在、まだTUIモードだと英語しか使えないので、`I want to translate this bash script to go.`と書いた上で↑のスクリプトを貼り付けました。
テストも、`Write down unit test.`と指示して書いてもらいました。

## 機能追加・修正

シェルスクリプトのときは`cd $todo_dir`で良かった`todo cd`ですが、Goから実行するときは動きません。  

```markdown
この case "cd" は cd サブコマンドの処理です。Goで実行するCLIは別プロセスなので、そこで cd しても親シェルの作業ディレクトリは変わりません。元のbashスクリプトは「関数」だったため cd $todo_dir がそのまま効きましたが、Goでは同じことはできないため、代わりに todoDir を表示して return しています。
```

<https://github.com/twpayne/chezmoi/blob/43ba0344d8465d916a006022a697e215204a90a9/internal/cmd/cdcmd.go>
`chezmoi cd`にインスパイアされた機能なので、chezmoiのコードをJunieに読ませて解説してもらいました。

```markdown
chezmoi cd の仕組みは「親シェルの cwd を変える」のではなく、「指定ディレクトリを cwd にしたサブシェルを起動する」です。コード上の流れはこうです。
• runCDCmd で CHEZMOI_SUBSHELL=1 をセット
• 引数がなければ source ディレクトリ、引数があれば解決してそのディレクトリを決定
• そのディレクトリが存在していてディレクトリであることを確認
• c.run(dir, cdCommand, cdArgs) でシェルを起動（cwd を dir にして起動）

cdCommand / cdArgs は設定で上書きできますが、未設定なら現在のユーザーシェル（shell.CurrentUserShell()）を使って起動します。つまり、
• chezmoi cd を実行すると source ディレクトリで新しいシェルが開く
• そのシェルを exit すると元の場所に戻る

という挙動です。親シェルの cwd は変わらないので、cd っぽい体験は「サブシェルを開いているから」実現されています。
```

なるほどなぁ...。  
確かに、`chezmoi cd`したあと、exitすると元のディレクトリに戻ってきます。  
この挙動でも特に不便そうな感じはしないので、同じ挙動にします。

```go
func runSubshell(dir string) {
    // Spawn the user's shell as a subshell in the target directory.
    shell := os.Getenv("SHELL")
    if shell == "" {
    shell = "sh"
    }
    cmd := exec.Command(shell)
    cmd.Dir = dir
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    _ = cmd.Run()
}
```

## まとめ

思いついてから使えるものが手に入るまでのリードタイムが短いとやる気が持続していいですね。  
とりあえず、しばらく使ってみようと思います。
