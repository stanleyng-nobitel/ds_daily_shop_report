---
name: Daily Store Report 基盤プロジェクト
description: TD→BQ→n8n→LINE WORKS の店舗ユーザー分析 + 日次レポート通知基盤の構築状況
type: project
---

TD（Treasure Data）に蓄積された店舗・ユーザーデータをBigQueryへ転送し、n8nで毎日集計してLINE WORKSにカード形式で通知する基盤。

**Why:** 非エンジニアが店舗別売上・来店数・新規会員数をLINE WORKSで毎朝確認できるようにするため。Looker Studioで深掘り分析も併用予定。

**How to apply:** 次の会話では未完了の前提条件の確認から始めること。

---

## BigQuery 設定（確定済み）

| 項目 | 値 |
|------|----|
| project_id | `n8n-td-claudecode` |
| dataset (raw) | `raw` |
| location | `asia-northeast1` |
| 認証方式 | Service Account |
| テスト用テーブル | `raw.test_table` |

---

## 作成済みワークフロー

- ワークフロー ID: `jhomsySSFmdWhzfH`
- ワークフロー名: `Daily Store Report → LINE WORKS`
- ノード数: 12（非アクティブ状態）
- BQ project ID は `n8n-td-claudecode` に設定済み

### フロー構成
```
[手動実行（テスト用）] または [毎朝9時実行（Schedule）]
 ↓
[BQ: daily_store_summary を取得]
  SQL: SELECT store_id, store_name, date, sales, visits, new_members,
            sales_diff_pct, visits_diff_pct, new_members_diff_pct
  FROM `n8n-td-claudecode.report.daily_store_summary`
 ↓
[Flex Message 生成]（Code node: 店舗別カード JSON を組み立て）
 ↓ (3並列)
[Merge: データ+Token] ← [Get line token（DataTable）]
 ↓
[Merge: 全データ合流] ← [Daily Report チャネルID（要設定）（Set node）]
 ↓
[リクエストボディ生成]（Code node: JSON.stringify）
 ↓
[LINE WORKS POST]（HTTP Request: POST to Bot API）
 ↓
[完了]
```

### LINE WORKS 通知形式
- Flex Message（カード形式）
- ヘッダー: 青背景 + 日付
- ボディ: 店舗名 / 売上 / 来店数 / 新規会員数 + 前日比（%、緑/赤色）

---

## BigQuery テーブル設計（未実行）

| レイヤー | テーブル/ビュー | 説明 |
|---------|--------------|------|
| raw | `raw.store_transactions` | TD BQ Export の出力先 |
| mart | `mart.daily_store_metrics` | 日次集計中間テーブル |
| report | `report.daily_store_summary` | n8n が参照するビュー（前日比付き） |

詳細 SQL: `/Users/n8n/.claude/plans/resilient-watching-map.md`

---

## 残り作業（ユーザー側）

| 項目 | 状態 |
|------|------|
| TD → BQ Export 設定 | 未完了 |
| n8n に Google BigQuery credential（SA）追加 | 未設定 |
| BQ ノードに credential 割り当て | 未設定 |
| mart / report テーブル SQL 実行 | 未実行 |
| LINE WORKS Daily Report チャネルID を Set ノードに入力 | 未設定 |
| 手動テスト実行 | 未実行 |
| Schedule Trigger 有効化 | 未実行 |

---

## 既存 LINE WORKS 連携パターン（流用元）

| 項目 | 値 |
|------|----|
| Sub-Workflow Line通知 | `SZyfRwL0wwTII6F5` |
| LINE WORKS Bot ID | `1420096` |
| line_token DataTable ID | `6iipvD4qo8LCSyrZ` |
| Google Sheets credential | `stAsixak35mlitqr` |
| Bot API エンドポイント | `https://www.worksapis.com/v1.0/bots/{botId}/channels/{channelId}/messages` |
