---
title: "Clineの利用コストを集計する"
emoji: "💸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Cline", "Deno"]
published: true
---

<!-- markdownlint-disable MD013 -->
[Claude Codeの使用料金を可視化するCLIツール「ccusage」を作った](https://zenn.dev/ryoppippi/articles/6c9a8fe6629cd6) に触発されて、Clineの利用金額を集計するツールを作りました。
<!-- markdownlint-enable MD013 -->
Mac以外では動作確認していないです。

[watarukura/clineusage](https://github.com/watarukura/clineusage)

```shell
./cline_cost.ts --from=20250501 --to=20250505
┌──────────┬──────────┬───────────┬─────────────┬────────────┬──────────┐
│ day      │ tokensIn │ tokensOut │ cacheWrites │ cacheReads │     cost │
├──────────┼──────────┼───────────┼─────────────┼────────────┼──────────┤
│ 20250503 │      160 │      4013 │       86728 │     560558 │ 0.493877 │
└──────────┴──────────┴───────────┴─────────────┴────────────┴──────────┘
```

Git cloneしない場合は↓こんな感じです。

<!-- markdownlint-disable MD013 -->
```shell
❯ deno run --allow-read --allow-env https://raw.githubusercontent.com/watarukura/clineusage/refs/heads/main/cline_cost.ts --from=20250510 --to=20250515
┌──────────┬──────────┬───────────┬─────────────┬────────────┬──────────┐
│ day      │ tokensIn │ tokensOut │ cacheWrites │ cacheReads │     cost │
├──────────┼──────────┼───────────┼─────────────┼────────────┼──────────┤
│ 20250510 │       55 │      6110 │      107113 │     509355 │ 0.646295 │
├──────────┼──────────┼───────────┼─────────────┼────────────┼──────────┤
│ 20250513 │       30 │      8703 │      180511 │     128667 │ 0.846151 │
└──────────┴──────────┴───────────┴─────────────┴────────────┴──────────┘
```
<!-- markdownlint-enable MD013 -->

`--from=YYYYmmdd`, `--to=YYYYmmdd` を指定しない場合は、全件が出力されます。
日毎に集計して出力する機能しかありません。
Denoで1ファイルのみで実装してあります。

## コスト計算

Clineが出力しているui_messages.jsonに出力されている値を足し合わせています。
<https://deepwiki.com/search/i-want-to-parse-uimessagesjson_9cbc8a8a-4e90-474d-964b-47c2196ffcdf>
deepWikiに質問してui_messages.jsonの構造を教えてもらいました。
自動生成されたドキュメントが質問に答えてくれる、ってすごい体験だなという感想です(語彙力なし)。
`type == "say" && say == "api_req_started"` の場合だけを考えればよいことがわかりました。
ChatGPTに頼んでシェルスクリプトを生成してもらって動作検証し、Denoで実行できるように書き換える、という形で実装しました。

単にシェルスクリプトに慣れてるのでこういうスタイルなんですが、TDDしづらいのでAI時代のコーディングスタイルとしては微妙かもです。

<!-- markdownlint-disable MD013 -->
```bash
#!/usr/bin/env bash
set -euo pipefail

from=${1:-"20020101"}
to=${2:-"29991231"}

BASE="$HOME/Library/Application Support/Code/User/globalStorage/saoudrizwan.claude-dev/tasks"
cd "$BASE" || exit 1

rm -f tmp.*

echo * |
    tr ' ' '\n' |
    grep -E "[0-9]{13}" |
    LANG=C sort |
    while read -r epoch_13; do
        epoch_10=${epoch_13:0:10}
        day=$(gdate --date="@$epoch_10" +%Y%m%d)
        echo "$epoch_13 $day" >>tmp.dirs
    done

find . -type f -name ui_messages.json |
    while read -r file; do
        dir=$(dirname $file | cut -c 3-15)
        day=$(grep -E "^$dir" tmp.dirs | cut -d' ' -f2)
        if [[ $from > $day ]] || [[ $to < $day  ]]; then
            continue;
        fi
        jq -r '.[] | select(.type == "say" and .say == "api_req_started" and .text != null) | .text' $file |
            jq -r --arg day "$day" '. | [$day, .tokensIn, .tokensOut, .cacheWrites, .cacheReads, .cost] | @tsv' >>tmp.tsv
    done

echo -e "day\ttokensIn\ttokensOut\tcacheWrites\tcacheReads\tcost"
LANG=C sort <tmp.tsv |
awk -F'\t' '
{
  day=$1;
  tokensIn=$2+0;
  tokensOut=$3+0;
  cacheWrites=$4+0;
  cacheReads=$5+0;
  cost=$6+0;
  sumTokensIn[day]+=tokensIn;
  sumTokensOut[day]+=tokensOut;
  sumCacheWrites[day]+=cacheWrites;
  sumCacheReads[day]+=cacheReads;
  sumCost[day]+=cost;
  OFS="\t" 
}
END {
  for (day in sumCost) {
    print day, sumTokensIn[day], sumTokensOut[day], sumCacheWrites[day], sumCacheReads[day], sumCost[day]
  }
}' | LANG=C sort

rm -f tmp.*
```
<!-- markdownlint-enable MD013 -->

## まとめ

Clineのチャット履歴の場所と読み方が雑にでもわかったので、他にも色々試せそうです。
<!-- markdownlint-disable MD013 -->
[gitleaks/gitleaks: Find secrets with Gitleaks 🔑](https://github.com/gitleaks/gitleaks) を使って、チャット履歴をスキャンしてシークレットがLLM呼び出しに含まれていないか早速チェックしてみたりしました。
<!-- markdownlint-enable MD013 -->
幸い、大丈夫そうでした。

<!-- markdownlint-disable MD013 -->
```shell
❯ gitleaks detect -v "$HOME/Library/Application\ Support/Code/User/globalStorage/saoudrizwan.claude-dev/tasks" --no-git

    ○
    │╲
    │ ○
    ○ ░
    ░    gitleaks

4:40PM INF scanned ~1890 bytes (1.89 KB) in 80.3ms
4:40PM INF no leaks found
```
<!-- markdownlint-enable MD013 -->
