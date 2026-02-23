---
title: "Google DocsをCLIでmarkdownに変換する"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Google Docs", "CLI"]
published: true
---

Google Calendarに「会議メモを作成」する機能があり、Google Docsをアタッチできるので便利に使っています。  
開発時に仕様の打ち合わせ内容をinputとしてAI Agentに開発をさせたいとき、Google Docsを読み込ませたいです。
MCPでGoogle Driveをどうにかする方法が見つからず、[gogcli](https://github.com/steipete/gogcli)も使ってみようと思ったのですがサードパーティのツールなので使用前によくよく調査が必要そうです。  
自前でシェルスクリプトを書いてやりたいことが実現できないか試してみます。

## 必要なもの

- [gcloud CLI](https://docs.cloud.google.com/sdk/docs/install-sdk?hl=ja)

## 作ったもの

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ ${1:-} == "" ]]; then
  echo "Google Docsのfile_idを指定してください"
  exit 1
fi
GOOGLE_DOCS_FILE_ID="$1"

gcloud auth login --enable-gdrive-access

curl -sL --get \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  --data-urlencode "id=$GOOGLE_DOCS_FILE_ID" \
  --data-urlencode "exportFormat=markdown" \
  "https://docs.google.com/feeds/download/documents/export/Export"
```

![img.png](https://github.com/watarukura/zenn-content/blob/main/articles/images/20260222_google_docs.pnghttps://github.com/watarukura/zenn-content/blob/main/articles/images/20241222_evidence.png?raw=true)
URLパスの`/d/`の後段から$FILE_IDを取得してください。  
↓こんな風に出力されます。

```bash
❯ ./google_docs_to_md.bash $FILE_ID
Your browser has been opened to visit:

    [$URL]

You are now logged in as $MAIL_ADDRESS.
Your current project is [$PROJECT_ID].  You can change this setting by running:
  $ gcloud config set project PROJECT_ID
こんにちは、世界。%
```

実行するとブラウザからGoogleにログインするよう求められます。  
ログインすると、credential情報が取得され、アクセストークンが発行できるようになります。  
アクセストークンをBearerトークンとしてHTTPヘッダにセットして呼び出すことで、Google Drive APIを呼び出すことができます。

## 以下、おまけ

認証がどのように行われるか、ADCでの認証との違いはなにか、などを調べてみました。

### `gcloud auth login --enable-gdrive-access` について

実行すると、下記のPATHにSQLite dbファイルが作成され、credential情報が書き込まれます。  
`--enable-gdrive-access` は、Google Drive APIへのアクセスを有効にします。

```sql
❯ sqlite3 ~/.config/gcloud/credentials.db "SELECT * FROM credentials;"
$MAIL_ADDRESS|{
  "client_id": "***********.apps.googleusercontent.com",
  "client_secret": "************************",
  "refresh_token": "*******************************************************************************************************",
  "revoke_uri": "https://oauth2.googleapis.com/revoke",
  "scopes": [
    "openid",
    "https://www.googleapis.com/auth/userinfo.email",
    "https://www.googleapis.com/auth/cloud-platform",
    "https://www.googleapis.com/auth/appengine.admin",
    "https://www.googleapis.com/auth/sqlservice.login",
    "https://www.googleapis.com/auth/compute",
    "https://www.googleapis.com/auth/accounts.reauth",
    "https://www.googleapis.com/auth/drive"
  ],
  "token_uri": "https://oauth2.googleapis.com/token",
  "type": "authorized_user",
  "universe_domain": "googleapis.com"
}
```

### ADCを利用してGoogle Calendarの情報も取得する

↑のcredentialを見れば分かる通り、Google DriveはScopeに含まれていますが、GMailやGoogle Calendarは含まれていません。  
これらのサービスのAPIを呼び出したい場合はApplication Default Credentials (ADC)を使用する必要があります。  
(Google Cloudでサービスアカウントを発行する方法もあります。)  
以下はADCを使用した場合のコマンド例です。

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ "${PROJECT_ID:-}" == "" ]]; then
  echo '$PROJECT_ID を指定してください'
  exit 1
fi
CALENDAR_ID="${CALENDAR_ID:-primary}" # primary = 自分のメインカレンダー
TZ="${TZ:-Asia/Tokyo}"
TODAY="$(TZ=$TZ date +%Y-%m-%d)"
TIME_MIN="${TIME_MIN:-${TODAY}T00:00:00+09:00}"
TIME_MAX="${TIME_MAX:-${TODAY}T23:59:59+09:00}"

gcloud config set project "$PROJECT_ID" >/dev/null
gcloud auth application-default login \
  --scopes="https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/calendar.readonly" >/dev/null
gcloud auth application-default set-quota-project "$PROJECT_ID" >/dev/null

curl -sS \
  -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
  -H "x-goog-user-project: ${PROJECT_ID}" \
  --get "https://www.googleapis.com/calendar/v3/calendars/${CALENDAR_ID}/events" \
  --data-urlencode "timeMin=${TIME_MIN}" \
  --data-urlencode "timeMax=${TIME_MAX}" \
  --data-urlencode "singleEvents=true" \
  --data-urlencode "orderBy=startTime" \
  --data-urlencode "maxResults=250" |
  jq -r '
    .items[] |
    select(.eventType != "workingLocation") |
    {
      title: .summary,
      start: .start.dateTime,
      end: .end.dateTime,
      attachment_file_id: .attachments[0].fileId // ""
    }
  '
```

なお、ADCの場合はJSONファイルとして以下にcredential情報が出力されます。  
<https://docs.cloud.google.com/docs/authentication/application-default-credentials?hl=ja>

```bash
❯ jq -r . ~/.config/gcloud/application_default_credentials.json
{
  "account": "",
  "client_id": "*********************************************.apps.googleusercontent.com",
  "client_secret": "************************",
  "quota_project_id": "****************",
  "refresh_token": "*******************************************************************************************************",
  "type": "authorized_user",
  "universe_domain": "googleapis.com"
}
```

## まとめ

画面からmarkdownエクスポートするのが面倒という怠惰な方向けです。  
↑のGoogle Calendar APIの呼び出しでは会議メモをGoogle Calendarから取得していますので、併用すればCLIで完結もできそうです。

## 蛇足、Google SpreadsheetをTSV出力する

ついでなので、Google SpreadsheetからTSV出力する場合は以下の通り。  

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ ${1:-} == "" ]]; then
  echo "Google Spreadsheetsのfile_idを指定してください"
  exit 1
fi
GOOGLE_SPREADSHEET_FILE_ID="$1"
GID="${2:-0}"

gcloud auth login --enable-gdrive-access

curl -sL --get \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  --data-urlencode "gid=$GID" \
  --data-urlencode "format=tsv" \
  "https://docs.google.com/spreadsheets/d/$GOOGLE_SPREADSHEET_FILE_ID/export"
```

![img.png](https://github.com/watarukura/zenn-content/blob/main/articles/images/20260222_google_spreadsheet.png?raw=true)
シートのIDをURLのクエリ文字列の`gid=`部分から取得してください。

```bash
❯ ./google_spreadsheet_to_tsv.bash $FILE_ID 0
# ...警告部分は省略
こんにちは      世界    ！%
```
