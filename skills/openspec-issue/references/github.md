# GitHub Issues / Projects の取り込み

GitHub を使う場合に読む。GitHub.com と GitHub Enterprise の両方を対象とし、ホスト、所有者、リポジトリ、Project 番号は接続先 URL から特定する。

以下の `<host>` などは、実行時に確認した値へ置き換える。**すべての gh 呼び出しに `GH_HOST=<host>` を指定する**。既定のホストに依存せず、本文やタイトルをシェルコードへ埋め込まない。

## チケット取得

```bash
GH_HOST="<host>" gh issue view "<number>" --repo "<owner>/<repo>" --json id,number,title,body,labels,state,url,updatedAt
GH_HOST="<host>" gh issue list --repo "<owner>/<repo>" --state open --limit 50 --json number,title,labels,url
```

取得できなければ、エラーと次の認証診断の結果を示して止まる。トークンを表示するオプションは使わない。

```bash
GH_HOST="<host>" gh auth status --hostname "<host>"
```

Project URL が入力されたら、項目の ID・種類・タイトル・リンク先を一覧で確認する。Issue へのリンクがあれば元の Issue を取得し、draft item ならその title / body を要求として読む。Pull Request や閲覧できない項目を Issue として扱わず、Project 番号を Issue 番号に使わない。

## Epic の子チケット

`epic` 指定またはプロジェクトで定めた Epic の種類・ラベルを確認した場合に使用する。

```bash
GH_HOST="<host>" gh api graphql --paginate --slurp \
  -H 'GraphQL-Features: sub_issues' \
  -f owner="<owner>" -f repo="<repo>" -F number="<number>" \
  -f query='
    query($owner: String!, $repo: String!, $number: Int!, $endCursor: String) {
      repository(owner: $owner, name: $repo) {
        issue(number: $number) {
          subIssues(first: 100, after: $endCursor) {
            nodes { number title state url repository { nameWithOwner } }
            pageInfo { hasNextPage endCursor }
          }
        }
      }
    }'
```

変数名は宣言・参照・引数でそろえる。`--slurp` はページの配列を返すので、各ページの `data.repository.issue.subIssues.nodes` をまとめる。子の labels / body は所属リポジトリから追加取得し、対象 PBI を選ぶ。関係は API の情報を優先し、本文のタスクリストだけで sub-issue と判断しない。Enterprise が API に対応していなければ、結果を推測しない。

## Project の情報と状態更新

共通手順の更新条件を満たす場合だけ、対象 Project の Status を変更する。`/opsx:project-status` が未導入でも、この手順で実行できる。

組織の Project URL なら以下を使う。ユーザー所有の Project なら `organization(login: $owner)` を `user(login: $owner)` に替える。

```bash
GH_HOST="<host>" gh api graphql --paginate --slurp \
  -f owner="<project-owner>" -F number="<project-number>" \
  -f query='
    query($owner: String!, $number: Int!, $endCursor: String) {
      organization(login: $owner) {
        projectV2(number: $number) {
          id title url
          fields(first: 100, after: $endCursor) {
            nodes {
              ... on ProjectV2SingleSelectField { id name options { id name } }
            }
            pageInfo { hasNextPage endCursor }
          }
        }
      }
    }'
```

返された URL を設定と照合し、Project ID、`Status` フィールドの ID、指定された状態の option ID を取得する。Status が別名ならプロジェクト設定で特定する。状態や選択肢を新しく追加しない。

Project の項目は次のクエリで取得する。入力が Project URL の場合の候補一覧にも使う。

```bash
GH_HOST="<host>" gh api graphql --paginate --slurp \
  -f projectId="<project-id>" \
  -f query='
    query($projectId: ID!, $endCursor: String) {
      node(id: $projectId) {
        ... on ProjectV2 {
          items(first: 100, after: $endCursor) {
            nodes {
              id isArchived
              content {
                __typename
                ... on Issue { id number title url repository { nameWithOwner } }
                ... on DraftIssue { id title body updatedAt }
                ... on PullRequest { id number title url }
              }
              fieldValueByName(name: "Status") {
                ... on ProjectV2ItemFieldSingleSelectValue { name optionId }
              }
            }
            pageInfo { hasNextPage endCursor }
          }
        }
      }
    }'
```

Status が別名なら、クエリのフィールド名も合わせる。全ページをまとめ、URL とリポジトリを照合して item ID を特定する。状態だけを更新する場合は、対象外の draft 本文を保存しなくてよい。選んだ項目の本文が必要なら追加取得できる。

対象が未登録、archived、閲覧不能のいずれかなら更新保留とする。自動で Project に追加したり archive を解除したりしない。目的の option ID なら更新を省き、完了扱いの状態から戻す場合は再着手の指示を確認する。

```bash
GH_HOST="<host>" gh project item-edit \
  --project-id "<project-id>" --id "<item-id>" \
  --field-id "<status-field-id>" --single-select-option-id "<option-id>"
```

更新後は同じ item の Status を再取得する。HTTP の成否に加え、GraphQL の `errors` と `data` も確認する。一部しか取得できていなければ完全な一覧として使わず、複数項目の更新結果は 1 件ずつ記録する。認証失敗や権限不足は診断を示し、別ホストや別の書き込み API で回避しない。

DraftIssue に通常の Issue URL がない場合は、Project URL と item ID を組にして出典とする。安定した項目リンクが取得できればそれも記録する。架空の Issue 番号・URL は作らない。

公式仕様: [gh api（変数・ページネーション）](https://cli.github.com/manual/gh_api)、[Issue の GraphQL フィールド](https://docs.github.com/en/graphql/reference/issues)、[Projects API](https://docs.github.com/en/issues/planning-and-tracking-with-projects/automating-your-project/using-the-api-to-manage-projects)。Enterprise では対象サーバーの対応状況も確認する。
