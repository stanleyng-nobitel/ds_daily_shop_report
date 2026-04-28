---
name: BQ直接アクセス方法
description: BigQueryへの直接アクセスはbq CLIツールとBashツールで可能。n8n経由は不要。
type: feedback
---

bq CLI（`/usr/local/share/google-cloud-sdk/bin/bq`）が認証済みアカウント（ben-dova@nobitel.co.jp）でインストール済みのため、Bashツールから直接BQクエリを実行できる。

**Why:** n8n経由の一時ワークフロー作成は不要で、`bq query --use_legacy_sql=false --project_id=n8n-td-claudecode 'SELECT ...'` で即座に確認できる。

**How to apply:** BQのデータ確認・検証・DDL実行はすべてBashツール経由でbq CLIを使う。n8nワークフロー経由は使わない。
