# openspec-extension-skills

チケット管理と OpenSpec の計画作成をつなぐ、追加スキルです。GitHub Issues / Projects、Jira などを対象に、既存の OpenSpec スキルと組み合わせて使います。

## スキル一覧

| スキル | 入力 | 行うこと |
| --- | --- | --- |
| [openspec-intake](skills/openspec-intake/SKILL.md) | 自由文の機能要望 | 既存チケット・仕様・実装と照合し、必要な PBI / Story・Epic の作成、追記、紐付けを行う |
| [openspec-issue](skills/openspec-issue/SKILL.md) | チケット番号・キー・URL | チケットを取り込み、既存の `openspec-propose` の手順で change の計画成果物を作る |

Intake はチケット整理まで、Issue は計画作成までを担当します。どちらも実装を自動で始めません。

## 導入

`skills/` 内の2つのディレクトリを、利用先の既存の `openspec-*` スキルと同じ場所に配置してください。スキル間の相対リンクは、この配置を前提にしています。

既存スキルが `<project>/.agents/skills/` にある場合は、次のように配置できます。`/path/to/project` は利用先のパスに置き換えてください。

```bash
git clone https://github.com/yoshi1220/openspec-extension-skills.git
cp -R openspec-extension-skills/skills/openspec-intake openspec-extension-skills/skills/openspec-issue /path/to/project/.agents/skills/
```

Issue には OpenSpec CLI と、同じ配置先の `openspec-propose` が必要です。計画の更新や実装には、既存の `openspec-update-change`、`openspec-apply-change` などを使います。Intake は、OpenSpec が未導入でも、参照できる資料を使ってチケットを整理できます。

チケットサービスへのアクセスには、実行環境の接続済みツール、認証済み CLI、公式 API を使います。他のサービスも対象にできますが、必要な取得・作成・紐付け操作を実行できる接続手段が必要です。

## 接続先の設定

接続先は、実行時に指定した URL、利用先プロジェクトの `AGENTS.md` / `CLAUDE.md` とその参照先、スキルの補足設定の順に確認します。小文字の `agents.md` / `claude.md` も扱います。

たとえば、利用先のプロジェクト指示に次のように記載できます。

```text
バックログ: https://example.atlassian.net/jira/software/projects/APP/boards/12
チケットの作成先プロジェクト: APP
```

GitHub ではリポジトリと追跡用 Project を、Jira ではプロジェクトとボードを区別して指定してください。チケットの種類・書式・階層も、利用先の規約に従います。

補足設定には、元のワークスペースの接続先候補が含まれています。利用先のプロジェクト指示で接続先を指定するか、[チケット接続先の補足設定](skills/openspec-issue/references/ticket-sources.md)を利用先に合わせて編集してください。接続先の記載とは別に、実行環境での認証が必要です。

Issue による着手状態の更新は、更新先・遷移先・タイミングが設定され、指示や運用で認められた場合に限ります。Intake はチケットや Project の状態を変更しません。

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

3. 計画をレビューし、既存のスキルで実装を始める。

   ```text
   $openspec-apply-change <change-name>
   ```

`$openspec-issue #7` やチケット URL の指定も使えます。`/opsx:intake`、`/opsx:issue` という依頼も受け付けますが、これらは OpenSpec CLI のサブコマンドではありません。

Intake は既存チケットを再利用し、実現済みの要望については新規作成を省きます。進行中の要求に影響する追記を行った場合は、次の apply 前に `$openspec-update-change <change-name>` で計画を同期します。

Issue は受け入れ条件（AC）の原文を保持し、Scenario・Task との対応を確認します。生成する成果物と順序は、選択した OpenSpec スキーマに従います。計画への対応付けは、実装による AC の達成とは区別して報告します。

実装後の確認と archive は、利用先の既存ワークフローに従ってください。archive による PR 作成やチケットの自動完了は前提にしていません。
