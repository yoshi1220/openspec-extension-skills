# チケット接続先の補足設定

接続先はプロジェクトの `AGENTS.md` / `CLAUDE.md` を優先する。小文字のファイル名を使う場合も同じ。このファイルは接続情報を補うもので、認証情報は書かない。

## このワークスペースから引き継いだ接続先

元のカスタムコマンドの接続先を残している。プロジェクト側に別の指定がない場合の候補とする。

```yaml
sources:
  - name: cat-github
    url: https://github.kddi.com/kag-ai-hacks-2026-summer/straight-toward-the-cat
    tracking_url: https://github.kddi.com/orgs/kag-ai-hacks-2026-summer/projects/5
    status_update:
      target: tracking
      value: In Progress
      timing: on-start
```

この接続先では、Issue ワークフローで仕様検討に着手した時点で Project の Status を変更する。接続先や状態を共通手順の固定値にはしない。Intake ではこの状態更新を実行しない。

## 他のプロジェクトでの設定

読み取り元は URL から特定する。複数サービスを使う場合は用途を分け、次の例のようにプロジェクト指示へ記載できる。例示の URL は実際の接続先候補に使わない。

```yaml
sources:
  - name: product-backlog
    url: https://example.atlassian.net/jira/software/projects/APP/boards/12
  - name: engineering-issues
    url: https://github.example.com/acme/catalog
    tracking_url: https://github.example.com/orgs/acme/projects/3
```

通常の文章で「バックログ: URL」「仕様検討の着手時は Status を『進行中』へ変更」のように記載してもよい。上記 YAML 形式を必須にはしない。

| 任意設定 | 用途 |
| --- | --- |
| `name` | 接続先を区別する呼び名。複数候補の選択にも使う |
| `url` | チケット、リポジトリ、バックログ、Project / board の URL |
| `tracking_url` | 要求の取得元と別の追跡先がある場合の URL |
| `provider` / `deployment` | URL で判別できないサービスや Cloud / Data Center の指定 |
| `acceptance_criteria_field` | 受け入れ条件を保持する専用フィールドの名前または実際の ID |
| `status_update.target` | チケット自身なら `ticket`、別の Project なら `tracking` |
| `status_update.value` | 実在する遷移先の状態名。実行時に確認して ID を解決する |
| `status_update.timing` | `none`（既定）、`on-start`（着手時）、`after-plan`（計画検証後） |

接続情報を記載しても、認証や操作権限が付与されるわけではない。利用する手段と権限は実行環境で確認する。チケット元と実装リポジトリの対応が分からなければ、それぞれ特定する。
