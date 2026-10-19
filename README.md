# openspec-extension-skills

機能要望のチケット化から、OpenSpec の計画作成・明確化までを支援する追加スキルです。GitHub Issues / Projects や Jira などから要求を取り込み、既存の OpenSpec スキルと組み合わせて使います。

## スキル一覧

| スキル | 入力 | 行うこと |
| --- | --- | --- |
| [openspec-intake](skills/openspec-intake/SKILL.md) | 自由文の機能要望 | 既存チケット・仕様・実装と照合し、必要な PBI / Story・Epic を作成・追記・紐付けする |
| [openspec-issue](skills/openspec-issue/SKILL.md) | チケット番号・キー・URL | チケットの要求を取り込み、既存の `openspec-propose` の手順で change の計画成果物を作る |
| [openspec-grill-me](skills/openspec-grill-me/SKILL.md) | 現在のアクティブな change・重点確認領域 | Propose / Issue 後、apply 前に存在する `spec`・`design`・`tasks` を選択式の質問で明確化する |

Intake はチケット整理、Issue は計画作成、Grill Me は既存計画の明確化を担当します。どのスキルも実装を自動で始めません。

## 導入

`skills/` 内の3つのディレクトリを、利用先の既存の `openspec-*` スキルと同じ場所に配置してください。スキル間の相対リンクを使うため、この配置が必要です。

既存スキルが `<project>/.agents/skills/` にある場合は、次のように配置できます。`/path/to/project` は利用先のパスに置き換えてください。

```bash
git clone https://github.com/yoshi1220/openspec-extension-skills.git
cp -R openspec-extension-skills/skills/openspec-intake \
  openspec-extension-skills/skills/openspec-issue \
  openspec-extension-skills/skills/openspec-grill-me \
  /path/to/project/.agents/skills/
```

Issue には OpenSpec CLI と、同じ場所に配置した `openspec-propose` が必要です。Grill Me には OpenSpec CLI と、計画成果物を持つ既存の change が必要です。Intake は OpenSpec が未導入でも、参照できる資料を使ってチケットを整理できます。

計画の更新や実装には、既存の `openspec-update-change`、`openspec-apply-change` などを使います。別のリポジトリにある Spec Kit のスキルには依存しません。

チケットサービスへのアクセスには、実行環境の接続済みツール、認証済み CLI、公式 API を使います。GitHub や Jira 以外のサービスも、必要な取得・作成・紐付け操作を実行できる接続手段があれば利用できます。

## 接続先の設定

接続先は、実行時に指定した URL、利用先プロジェクトの `AGENTS.md` / `CLAUDE.md` とその参照先、スキルの補足設定の順で確認します。小文字の `agents.md` / `claude.md` も扱います。

たとえば、利用先のプロジェクト指示に次のように記載できます。

```text
バックログ: https://example.atlassian.net/jira/software/projects/APP/boards/12
チケットの作成先プロジェクト: APP
```

GitHub ではリポジトリと追跡用 Project を、Jira ではプロジェクトとボードを区別して指定してください。チケットの種類・書式・階層も、利用先の規約に従います。

補足設定には、元のワークスペースの接続先候補が含まれています。利用先のプロジェクト指示で接続先を指定するか、[チケット接続先の補足設定](skills/openspec-issue/references/ticket-sources.md)を編集してください。接続先の指定に加えて、実行環境での認証も必要です。

Issue が着手状態を更新するのは、更新先・遷移先・タイミングが設定され、指示や運用で認められた場合に限ります。Intake はチケットや Project の状態を変更しません。

## 使い方

Codex では、次のように依頼します。チケットが既にあれば Intake を省略できます。

1. 要望をチケットに整理する。

   ```text
   $openspec-intake 講義ページに合計時間を表示したい。ステップに休憩も入れたい。
   ```

2. 返されたキーを指定し、計画を作る。

   ```text
   $openspec-issue APP-123
   ```

3. 選択式の質問で計画を明確にし、成果物間の矛盾や受け入れ条件の抜けを解消する。

   ```text
   $openspec-grill-me
   ```

4. 計画の確認を終えたら、既存のスキルで実装を始める。

   ```text
   $openspec-apply-change <change-name>
   ```

チケットの指定には、`$openspec-issue #7` やチケット URL も使えます。`/opsx:intake`、`/opsx:issue` という依頼も受け付けますが、OpenSpec CLI のサブコマンドではありません。

Intake が確認する既存チケットは、クローズされていないものに限ります。Epic・親子・関連チケットにも同じ条件を適用します。GitHub では Issue 自体の `OPEN` / `CLOSED`、Jira などではプロジェクト規約とワークフローで判定します。クローズ済みチケットの本文は調べず、重複照合も対象範囲内のクローズされていないチケットに限ります。

仕様・実装とも照合し、実現済みの要望についてはチケットの新規作成を省きます。進行中の要求に影響する追記をした場合は、次の apply 前に `$openspec-update-change <change-name>` で計画を同期します。

Issue は受け入れ条件（AC）の原文を保ち、Scenario・Task との対応を確認します。生成する成果物と順序は、選択した OpenSpec スキーマに従います。AC を計画に対応付けたことと、実装によって AC を満たしたことは区別して報告します。

Grill Me の対象は、直前の `openspec-propose` / `openspec-issue` で作成・選択したアクティブな change 1 件が基本です。会話で特定できず、アクティブな change が1件だけなら、その change を選びます。対象が曖昧な場合だけ選択を確認します。

レビュー・明確化するのは、apply 前に存在する `spec`・`design`・`tasks` だけです。存在しないファイルは対象外とし、proposal は読み取り専用で参照します。実装開始済みの change や完了済みタスクは見直しません。

指定された store とスキーマは維持します。Issue 由来の計画では AC の原文と ID を保ち、存在する対象ファイル間で Scenario・Task の対応を検証します。質問や追加記述は日本語を基本とし、OpenSpec の構造キーワードは変えません。

実装後の確認と archive は、利用先の既存ワークフローに従ってください。archive に伴う PR 作成やチケットの自動完了は想定していません。
