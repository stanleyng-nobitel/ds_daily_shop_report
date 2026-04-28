# Memory Index

- [User Profile](user_profile.md) — n8n運用担当、BQ・LINE WORKS・Google Sheets連携多数あり
- [n8n接続情報](reference_n8n_connection.md) — URL・APIキー・手動実行方法・主要ワークフローID（3本）・credential ID
- [日次予算スプレッドシート](reference_budget_sheet.md) — ID: 14YhcFvjToD7esslEjQrTb4a12NKz67t9-9Avg96wRT8、予算（日次）シート、gws CLIでアクセス可
- [BQ直接アクセス方法](feedback_bq_direct_access.md) — bq CLIでBashから直接クエリ実行可能。n8n経由不要。
- [KPI定義変更時はn8nも必ず同期](feedback_definition_sync.md) — 定義変更時はn8n jhomsySSFmdWhzfH の2つのCodeノードも更新必須
- [KPI計算定義（周期・アクティブ率）](project_kpi_definitions.md) — 周期=来店数/ユニーク客数、アクティブ率=ユニーク客数/会員数。BQカラム名付き
- [34期・33期グループマッピング（確定版）](project_group_mapping.md) — 既存=33期既存+既存新店、既存新店=33期新店（FY25）、新店=対応なし。prev_day・yoy CTE実装付き
- [result_33_daily テーブル設計と注意点](project_result33_daily.md) — stretch_salesは消化売上相当。月合計列は分母用でSUM禁止
- [運用・データフロー設計](operation.md) — TD→BQ→n8nフロー、BQビュー構成、メールレポート項目一覧、budget_34_monthly使用確定、前年表示実装済み
- [LINE WORKS チャットbot 開発プラン](lineworks_bot_plan.md) — Google Sheets→n8n→MCP(Claude+BQ)構成、セキュリティ設計（未実装）
