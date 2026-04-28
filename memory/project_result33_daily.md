---
name: result_33_daily テーブル設計と注意点
description: 33期日次実績BQテーブルの構造・ワークフロー・月合計列の扱いに関する注意点
type: project
originSessionId: 023b5a16-1f16-4340-9c37-e6a68924ec6e
---
## BQテーブル: result_33_daily

作期実績（日次）シートから33期日次実績をBQに格納するテーブル。

**データソース:** Google Sheets「作期実績（日次） 」（末尾スペースあり）
**Spreadsheet ID:** `14YhcFvjToD7esslEjQrTb4a12NKz67t9-9Avg96wRT8`

**n8nワークフロー:** `AKknyVHi4elyNJcE`（Result 33 Sheet to BQ result_33_daily）
- 毎日6:00実行、TRUNCATE → 全件INSERT
- 6月・7月など月追加時はシート側に列を追加するだけで自動対応（ワークフロー修正不要）

**テーブル構造（1行 = 1店舗 × 1日）:**
| カラム | 型 | 内容 |
|---|---|---|
| group_33 | STRING | 33期グループ（既存/既存新店/新店（FY25）） |
| shop_id | INT64 | 店舗ID |
| shop_name | STRING | 店舗名 |
| am | STRING | AM名 |
| region | STRING | 地域 |
| date | DATE | 日付（2026-04-01〜） |
| month | STRING | 月（2026-04 / 2026-05 ...） |
| stretch_sales | FLOAT64 | 消化売上（日次）※カラム名はstretch_salesだが実態は消化売上（consumed_based_sales相当） |
| new_visitors | FLOAT64 | 新規客数（日次） |
| new_members | FLOAT64 | ②会員制_入会者数(新規)（日次） |
| stretch_sales_monthly | FLOAT64 | 消化売上（月合計）※同上 |
| new_visitors_monthly | FLOAT64 | 新規客数（月合計） |
| new_members_monthly | FLOAT64 | ②会員制_入会者数(新規)（月合計） |
| updated_at | TIMESTAMP | 更新日時 |

**件数:** 153店 × 61日（4〜5月）= 9,333件

## 月合計列（_monthly）の扱いに関する重要注意点

**Why:** `stretch_sales_monthly`等の月合計列は、**同じ月の全日に同じ値が繰り返し格納されている**。
これはdaily÷monthly比率を計算するための分母として使うため。

**How to apply:**
- `SUM(stretch_sales_monthly)` は絶対NG → 30日分が掛け算されて30倍になる
- 正しい使い方: `stretch_sales / stretch_sales_monthly AS daily_ratio`（当日の日次÷月合計）
- 既存レポート（jhomsySSFmdWhzfH / v_daily_kpi_with_budget）はこのテーブルを参照していないため現時点では影響なし

## budget_34_daily への月合計追加（2026-04-14）

`budget_amount_monthly` 列を`budget_34_daily`に追加。

**Why:** 予算シートの「4月（合計）」等も日次÷月合計の分母として使うため。6月・7月追加時も自動対応。

**How to apply:**
- `v_daily_kpi_with_budget`は`budget_amount`のみ参照しており、`budget_amount_monthly`は非参照 → 既存レポートへの影響なし
- 新しいカラムを追加・既存カラム値は変更なし・result_33_dailyも新規のみ → 今回の変更は全て既存レポートに影響なし
- 月合計列はSUM禁止、分母専用で使うこと（budget_34_daily・result_33_daily共通ルール）
