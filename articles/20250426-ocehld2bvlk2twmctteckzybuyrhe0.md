---
title: "Confluence Serverにmarkdownドキュメントを同期したい"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Confluence"]
published: true
---

現在の勤め先ではオンプレミスのConfluence Serverを使用しています。  
<https://github.com/markdown-confluence/markdown-confluence> を使用してGitHub上のドキュメントを同期したいと考えて調査しました。  

## Confluence Storage Format

[Confluence 保存形式 | Confluence Data Center 9.4 | アトラシアン製品ドキュメント](https://ja.confluence.atlassian.com/doc/confluence-storage-format-790796544.html)
Confluence Serverでは、Confluence Storage FormatというXHTMLベースのフォーマットで保存されます。  
REST APIを使って、storage形式で出力することもできます。  
body部分だけ抜き出したHTMLのような奇妙な形式になります。

```shell
curl -sL "$url/rest/api/content/$page_id?expand=body.storage" \
        --header 'Accept: application/json' \
        --header "Authorization: Bearer $token" \
        >tmp.json
```

```shell
cat tmp.json | jq -r .body.storage.value
<p class="auto-cursor-target"><br /></p><table><tbody><tr><th><p>1</p></th><th><p>2</p></th><th><p>3</p></th></tr><tr><td><p>foo</p></td><td><p>bar</p></td><td><p><br /></p></td></tr></tbody></table><p class="auto-cursor-target"><br /></p>```
```

試してみたところ、[microsoft/markitdown: Python tool for converting files and office documents to Markdown.](https://github.com/microsoft/markitdown) を使うとmarkdownに変換できました。

```shell
❯ jq -r .body.storage.value <tmp.json |
      uvx "markitdown[all]"
| 1 | 2 | 3 |
| --- | --- | --- |
| foo | bar |  |
```

## マークアップでの挿入

記事の編集画面でMacの場合はCmd + Shift + Dで「マークアップを挿入」が開きます。
[!img.png](images/20250426_confluence_markup_insert.png)

markdownまたはConfluence Wiki形式で記事中に「マークアップを挿入」が可能です。
[Confluence Wiki Markup | Confluence Data Center 9.4 | Atlassian Documentation](https://confluence.atlassian.com/doc/confluence-wiki-markup-251003035.html)

が、REST APIでmarkdown形式でpage contentを作成する際、マークアップの方式を選択できません。  
そのためか、markdown形式での作成ではテーブルなどが崩れてしまうため、Confluence Wiki形式での作成がオススメです。
[Solved: Insert Confluence / Wiki Markdown via API](https://community.atlassian.com/forums/Confluence-questions/Insert-Confluence-Wiki-Markdown-via-API/qaq-p/667936)

[Shogobg/markdown2confluence: Tool to convert Markdown to Confluence wiki](https://github.com/Shogobg/markdown2confluence) を使うと、markdownからconfluence Wikiへの変換が可能です。
ただ、テーブルの表示がやはり崩れてしまうため、renderer optionを指定しています。  
以下のIssueを上げていますが対応してもらえるといいなぁ...。
[Markdown table syntax little change · Issue #15 · Shogobg/markdown2confluence](https://github.com/Shogobg/markdown2confluence/issues/15)

ざっくりこんな感じで変換してpage作成ができます。

```shell
npx @shogobg/markdown2confluence test.md > test.txt
content=$(cat test.txt)

content_json=$(jq -n \
    --arg title "$title" \
    --arg content "$content" \
    --arg space "$space" \
    --arg id "$parent_page_id" \
    '{
        type: "page",
        title: $title,
        body: {
            storage: {
                value: $content,
                representation: "wiki"
            }
        },
        space: $space,
        ancestors: [{"id": ($id | tonumber)}]
    }')

curl -X POST -sL "$url/rest/api/content" \
    --header 'Accept: application/json' \
    --header 'Content-Type: application/json' \
    --header "Authorization: Bearer $token" \
    -d "$content_json"
```

## ADF: Atlassian Document Format

Confluence CloudではADFというJSONを使用した形式で記事が保存されています。  
[On-Premise Confluence Server Support · Issue #91 · markdown-confluence/markdown-confluence](https://github.com/markdown-confluence/markdown-confluence/issues/91) によると、markdown-confluenceではADFのみに対応しているため、Confluence Serverでは利用できないようです。  
悲しい...。Issueでは有志が対応しようとしているので期待していますが、以下の2つの問題があって未解決のようです。  

1. markdownからConfluence storage formatへの変換
2. Personal Access Tokenでの認証方式がConfluence CloudとConfluence Serverで異なる

## まとめ

Confluence Serverからダウンロードしてmarkdownに変換、はできましたし、markdownからConfluence Wiki形式に変換してアップロード、もできました。
が、GitHubでmarkdown形式のドキュメントを書いてConfluence Serverと同期したい、というところまではまだ何段階か課題が残っています。  
この記事が同じ悩みを持つ方の参考になれば幸いです。
