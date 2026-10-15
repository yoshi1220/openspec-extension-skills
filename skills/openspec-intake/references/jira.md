# Jira での Intake

Jira を使う場合に読む。[共通の Jira 取得手順](../../openspec-issue/references/jira.md) から、接続先の特定、認証、検索、ADF と親子関係の取得を使う。**ワークフロー遷移は実行しない**。特定の Jira CLI や MCP ツール名は前提にしない。

## 作成・編集前の確認

- サイト、作成先 project、Epic / Story / PBI 相当の issue type、階層、AC 専用フィールド、必要な作成フィールドを特定する。board ID と作成先 project は同一ではない。
- プロジェクト規約とワークフローでクローズ扱いの状態を確認し、それを除く条件を JQL に付ける。完了カテゴリをクローズ扱いにする環境なら、`project = APP AND statusCategory != Done` のように検索する。Epic・親子・関連チケットの検索にも同じ条件を付け、`Closed` という名前だけで判定しない。
- クローズされていない範囲を、Cloud の `nextPageToken`、Data Center の対応するページネーション等で最後まで取得する。検索範囲を絞った場合は、その範囲でしか重複照合していないことを示す。クローズ済みチケットが混ざっても、本文の調査や追加取得はしない。
- 対象チケットの最新の本文・AC・状態・resolution・親・更新日時を読む。実現済みかどうかは、進行中の change や仕様・実装の根拠に照らして判断する。親がクローズ済みなら、その本文・子一覧は取得しない。
- 接続先 API / ツールで、作成時の必須フィールドと編集可能なフィールドを確認する。必要な役割、種別、親や権限が不明なら、書き込み前に解決する。管理設定、Issue type、custom field 自体は作成しない。

Cloud REST v3 の読み取り先は次のとおり。接続済みクライアントの API ベース URL からの相対パスを示す。

```text
GET /rest/api/3/issue/createmeta/{projectIdOrKey}/issuetypes
GET /rest/api/3/issue/createmeta/{projectIdOrKey}/issuetypes/{issueTypeId}
GET /rest/api/3/issue/{issueIdOrKey}?expand=names,schema
GET /rest/api/3/issue/{issueIdOrKey}/editmeta
```

メタデータ一覧も必要に応じてページを送る。Data Center は対象サーバーの API とメタデータ形式を使う。

## 新規 Epic / PBI の作成

Cloud の基本操作は `POST /rest/api/3/issue`。実際の project・issue type・作成可能フィールドを使い、[標準書式](ticket-format.md) の内容をサービスの保存形式へ対応付ける。

```json
{
  "fields": {
    "project": {"key": "<project-key>"},
    "issuetype": {"id": "<confirmed-issue-type-id>"},
    "summary": "<title>",
    "description": {
      "type": "doc",
      "version": 1,
      "content": []
    }
  }
}
```

上の例は送信データの構造を示すもので、空の `content` のまま登録しない。description や複数行の専用フィールドが ADF を要求する場合は、見出し・箇条書き・条件を対応するノードに変換する。JSON は文字列連結で作らず、必須フィールドは実データか確認済みの既定値で埋める。優先度・担当者・公開範囲等を独断で選ばない。

Epic を先に作り、返された実際のキーを PBI の親に使う。環境の階層と作成メタデータで `parent` が使える場合は、PBI の作成 fields に `"parent": {"key": "<epic-key>"}` を加えられる。旧 Epic Link や別の関連フィールドが必要な場合は、実際のフィールド ID を取得して使う。Epic 配下の通常 Story を subtask type に変えて代用しない。

返された key / id でチケットを再取得し、閲覧 URL と親を確認する。作成応答が不明なら、検索で反映を確かめるまで create を再送しない。検索の反映遅延もありうるため、1 回の検索で見つからないことを未作成の証拠にしない。

## 追記と親子の紐付け

Cloud の基本操作は `PUT /rest/api/3/issue/{issueIdOrKey}`。直前の読み取りを基に、変更する本文・専用 AC フィールドだけを送る。既存 ADF ノード、マーク、チェック状態と条件の順序を保持する。

- 追記・紐付けの直前に、対象チケットと親の状態を再確認する。クローズ済みに変わっていたら、そのチケットに関わる書き込みを止める。
- AC が専用フィールドなら、その末尾へ追記する。description にだけ追記して専用フィールドが古いままにならないようにする。
- 親が同じなら変更せず、親がない場合にだけ、対応するネイティブフィールドで紐付ける。既存親の移動は Intake の承認範囲に含めない。
- Epic 一覧や関連チケットの備考も、最新内容に追加してから再取得する。ツール / API が保証しない原子的な更新やバージョン検査を想定しない。
- 本文更新の payload に `transition`、コメント、既存ラベル・担当者の変更を含めない。`/transitions` は呼ばない。必要な親子関連以外の issue link を無関係に追加しない。

## Board / backlog と結果

Jira board に表示されるかはフィルタで決まる。GitHub Project の item-add と同じ登録 API があるとは想定せず、作成先 project と既存フィルタで対象 board に表示されるかを確認する。表示のために Status・ラベル・Sprint・board filter を変更しない。

親子関係・本文・AC と、必要な追跡先での扱いを確認してから成功を報告する。一部が未対応 / 未反映ならチケットの作成成功と分ける。返すのは Epic と PBI の実際のキーであり、board 番号や内部の数字 ID を代用しない。

公式仕様: [JQL の状態・解決状況フィールド](https://support.atlassian.com/jira-software-cloud/docs/jql-fields/)、[Cloud issue の作成・編集・メタデータ](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/)、[Cloud fields](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-fields/)、[Cloud 検索](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-search/)、[Data Center メタデータと作成の例](https://developer.atlassian.com/server/jira/platform/jira-rest-api-examples/)。
