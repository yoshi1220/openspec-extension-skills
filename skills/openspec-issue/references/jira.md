# Jira の取り込み

Jira を使う場合に読む。接続済みの Jira / Atlassian ツールがあれば、その機能・認証・ページネーションに従う。なければ設定済みの CLI か公式 REST API を使い、特定の Jira CLI の存在は前提にしない。

## 接続先と ID

- チケット URL、サイト URL、Project / board URL から対象を特定する。board ID、project key、内部 issue ID、`APP-123` の issue key は区別する。
- サイト URL だけで複数プロジェクトが候補になる場合は選択を求める。`123` を `APP-123` に補うのはプロジェクトが一意に決まる場合だけ。
- Jira Cloud と Data Center の種類、利用する API のバージョンは接続情報やサーバー情報から確認する。自社ドメインだからという理由だけで Data Center と決めつけない。
- API のベース URL には構成に応じたコンテキストパスが含まれる場合がある。OAuth で `api.atlassian.com/ex/jira/<cloudId>` を使う接続と、サイト URL へ直接アクセスする接続も区別する。
- 既存の認証設定を使い、トークンをファイルや成果物へ書かない。認証がなければ必要な接続を示して止まる。取得権限のない本文を公開ページや検索結果から補わない。

## 要求の取得

Jira Cloud REST v3 では次の操作を使う。URI は接続済みクライアントのベース URL からの相対パスを示す。

```text
GET /rest/api/3/issue/{issueIdOrKey}?expand=names,schema
GET /rest/api/3/field
GET /rest/api/3/search/jql?jql={URLエンコードしたJQL}
```

- issue の `key`、`summary`、`description`、`issuetype`、`status`、`updated`、親子関係、受け入れ条件を取得する。正規の閲覧 URL はサイトの `/browse/{key}` 等から解決し、REST API の `self` を閲覧 URL として使わない。
- 受け入れ条件の custom field ID は環境依存。プロジェクト設定、`names` / `schema`、フィールド一覧で実際のフィールドを特定し、`customfield_100xx` を固定しない。
- Cloud の description などが Atlassian Document Format（ADF）なら、見出し、リスト、チェック項目、表、リンク、コードを保って読み取る。単純な `text` の連結で条件を欠落させない。原文が専用フィールドにある場合はそこを出典とする。
- Epic の子と通常の subtask は区別する。対応する環境では `parent = APP-100` の検索を使う。古い Epic Link のカスタムフィールドなどを使う環境では、実際の階層設定を確認して取得する。
- 一覧や Epic の子は、API が返す `nextPageToken` や `isLast` 等に従って最後のページまで取得する。途中のページを全件とは扱わず、Cloud の古い `/search` だけにも依存しない。
- Data Center では対応する REST v2 と、そのサーバーの検索・階層・ページネーション仕様を使う。Cloud v3 や ADF をそのまま前提にしない。

## 着手状態への遷移

共通手順の状態更新条件を満たし、`target: ticket` が設定された場合だけ実行する。Jira board の列名は status や transition 名と一致するとは限らない。

```text
GET /rest/api/3/issue/{issueIdOrKey}/transitions?expand=transitions.fields
POST /rest/api/3/issue/{issueIdOrKey}/transitions
```

実行可能な遷移と遷移先の status を調べ、設定に一致する transition ID を選ぶ。基本の送信内容は次の形になる。

```json
{"transition":{"id":"実際に取得した遷移ID"}}
```

- 同じ遷移先への transition が複数ある場合や、必須フィールドが不足する場合は、既存設定で解決できるか確認する。解決できなければ更新保留とし、担当者やコメントを推測で補わない。
- すでに目的の状態なら遷移しない。Done / Closed 等からの再着手はユーザーの指示を確認する。
- `PUT issue` で status を直接書き換えようとしない。遷移後は issue の status を再取得する。
- `target: tracking` の場合は、その board / 外部追跡先が持つ実際の更新方法を確認する。別のワークフロー遷移を代用しない。
- 認証・取得失敗時は取得を止めて事実を報告する。遷移だけが失敗した場合は更新前の状態と更新保留を残す。無条件の再試行や巻き戻しはしない。

公式仕様: [Cloud issue の取得・遷移](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/)、[Cloud issue fields](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-fields/)、[Cloud 検索](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-search/)、[Data Center issue API](https://developer.atlassian.com/server/jira/platform/rest/v10005/api-group-issue/)。Data Center は対象サーバーに対応する版を確認する。
