# zenn-content

zenn記事公開用リポジトリ

## 記事を書く

```bash
npm run new:article
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
npm run lint
```

自動修正されない場合は手で直すこと。

```bash
npm run fix
```

## Thanks

- https://github.com/kufu/textlint-rule-preset-smarthr
- https://zenn.dev/serima/articles/4dac7baf0b9377b0b58b