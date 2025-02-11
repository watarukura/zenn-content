# zenn-content

zenn記事公開用リポジトリ。

## 必要なツール

- aquaproj/aqua
  - `brew install aquaproj/aqua/aqua` でinstall
- lefthook
  - aquaをinstallしたあと、初回のみ`lefthook install`が必要
- pnpm
  - node.jsはaquaでinstallされるので、`corepack enable pnpm`する

## 記事を書く

```bash
pnpm new:article
```

VS Codeで書く。

## 推敲する

Pull Requestを作るとGitHub Actionsでtextlintする。

```bash
git switch -c {branch_name}
git add .
git commit -m "comment"
git push origin HEAD
gh pr create
gh pr checks
```

ローカル実行してもよい。

```bash
pnpm lint
```

自動修正されない場合は手で直すこと。

```bash
pnpm fix
```

Markdownとしての形式もlint/fixできる。

```bash
pnpm lint:md
pnpm fix:md
```

## Thanks

- <https://github.com/kufu/textlint-rule-preset-smarthr>
- <https://zenn.dev/serima/articles/4dac7baf0b9377b0b58b>
