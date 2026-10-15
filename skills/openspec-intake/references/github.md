# GitHub での Intake

GitHub を使う場合に読む。[共通の GitHub 取得手順](../../openspec-issue/references/github.md) から、ホストの特定、認証診断、Epic の子と Project の読み取りを使う。**Status 更新操作は実行しない**。ホスト、所有者、リポジトリ、Project 番号は設定から決め、すべての `gh` 呼び出しに `GH_HOST=<host>` を指定する。

以下の番号・ホスト等は、確認した値へ置き換える。タイトル等の自由文は、構造化引数か安全な CLI 引数配列で渡す。

## 書き込み前の照合

全状態の一覧は、ページ上限で切れない方法で取得する。例として REST の結果を専用一時領域へ保存する。

```bash
GH_HOST="<host>" gh api --paginate --slurp \
  "repos/<owner>/<repo>/issues?state=all&per_page=100" \
  > "<temporary-directory>/issues.json"
```

REST の一覧には PR も含まれるため、`pull_request` を持つ要素は候補 PBI から除く。`--slurp` の結果は全ページをまとめる。番号・タイトル・状態・`state_reason`・ラベル・URL・更新日時で候補を絞って本文を読み、途中のページ取得に失敗しても「残りなし」とは扱わない。

```bash
GH_HOST="<host>" gh issue view "<number>" --repo "<owner>/<repo>" \
  --json id,number,title,body,labels,state,url,updatedAt
GH_HOST="<host>" gh api --paginate --slurp \
  "repos/<owner>/<repo>/labels?per_page=100"
```

新規 PBI / Epic に付ける `pbi`、`epic`、`role:*` 等は、プロジェクト規約と実在するラベルを照合する。存在しないラベルを作ったり、既存チケットのラベルを変更したりしない。ラベルではなく Issue type を使う運用なら、実際のスキーマと対応ツールで扱う。必要な分類を無断で省略しない。

Epic の子と Project 項目は共通の取得手順で全ページ読む。既存 PBI の親も確認する。

```bash
GH_HOST="<host>" gh api graphql \
  -H 'GraphQL-Features: sub_issues' \
  -f owner="<owner>" -f repo="<repo>" -F number="<number>" \
  -f query='
    query($owner: String!, $repo: String!, $number: Int!) {
      repository(owner: $owner, name: $repo) {
        issue(number: $number) {
          id number url updatedAt
          parent { id number url }
        }
      }
    }'
```

GraphQL に `errors` がある場合や、repository / issue 自体が null の場合は、親子関係がないとは判断しない。エラーなく対象 issue を取得でき、`parent` だけが null なら、親が未設定と判断できる。Enterprise で API が異なる場合は、そのサーバーの正式な同等 API を確認する。必要な機能が使えなければ紐付けは未対応とし、本文リンクだけで成功とは報告しない。

## 新規作成と追跡先への登録

新規 Epic と PBI は [標準書式](ticket-format.md) またはプロジェクト書式で本文を用意する。作成直前にも今回の要望と一致する既存チケットがないか確認する。

```bash
GH_HOST="<host>" gh issue create --repo "<owner>/<repo>" \
  --title "<title>" --body-file "<temporary-body-file>" \
  --label "<confirmed-label>"
```

`--label` は確認できた必要な新規ラベルの数だけ指定し、不要なら省略する。返された URL から Issue を再取得し、number と GraphQL node ID を記録する。作成応答が不明なら、近い時刻・タイトル・本文等で照合してから判断し、同じ create を再送しない。

設定された Project へ登録する。まず URL / Issue ID で既存項目を照合し、登録済みなら省略する。対象 Project はタイトルで曖昧に選ばず、URL の owner と番号で特定する。

```bash
GH_HOST="<host>" gh project item-add "<project-number>" \
  --owner "<project-owner>" --url "<issue-url>" --format json
```

Intake では `gh project item-edit` や Status 変更の mutation を呼ばない。新規登録時の既定値や Project 自動化による状態はサービス側の動作として確認・報告し、意図的な状態変更操作と混同しない。

GitHub Project URL しかなく作成先リポジトリが未確定なら、Issue を作る前に特定する。DraftIssue を選ぶ運用が明示されていなければ、Issue の代わりに draft item を作って番号を捏造しない。

## Epic の sub-issue として紐付ける

親と子の URL / node ID、現在の親を照合する。同じ親なら省略し、別の親がある場合は自動で移動しない。親がない場合だけ、確認済み node ID で追加する。

```bash
GH_HOST="<host>" gh api graphql \
  -H 'GraphQL-Features: sub_issues' \
  -f parentId="<epic-node-id>" -f childId="<pbi-node-id>" \
  -f query='
    mutation($parentId: ID!, $childId: ID!) {
      addSubIssue(input: {issueId: $parentId, subIssueId: $childId}) {
        issue { id number url }
        subIssue { id number url }
      }
    }'
```

`replaceParent: true` は指定しない。Issue 番号、REST の整数 ID、GraphQL node ID を混同しない。反映後は子の親と、親の子一覧を再取得して照合する。

## 既存本文への追記

編集直前に `gh issue view` で最新本文・状態・更新日時を再取得し、追記先が未完了か、計画同期が必要かも再評価する。最新本文を保持したまま必要な行だけを加え、一時ファイルへ保存する。

```bash
GH_HOST="<host>" gh issue edit "<number>" --repo "<owner>/<repo>" \
  --body-file "<temporary-body-file>"
```

この操作は本文全体を置き換えるため、古い本文を使わない。`gh issue edit` が同時更新を確実に検出するとは想定せず、更新後に追記と既存内容の保持を確認する。Epic の一覧や関連チケットの追記も同じ手順で行う。コメント、既存ラベル、担当者、状態を変更するオプションは付けない。

公式仕様: [gh api](https://cli.github.com/manual/gh_api)、[gh issue create](https://cli.github.com/manual/gh_issue_create)、[gh issue edit](https://cli.github.com/manual/gh_issue_edit)、[Project 登録](https://cli.github.com/manual/gh_project_item-add)、[GraphQL addSubIssue / parent](https://docs.github.com/en/graphql/reference/issues)。Enterprise の対応状況は実行時に確認する。
