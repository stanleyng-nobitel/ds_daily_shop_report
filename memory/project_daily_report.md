---
name: Daily Store Report 基盤プロジェクト
description: TD→BQ→n8n→LINE WORKS / Gmail の店舗KPI日次レポート通知基盤の構築状況
type: project
---

BQに蓄積された店舗・ユーザーデータをn8nで毎日集計し、LINE WORKSにFlex Message（Carousel）+ GmailにHTMLメールを送信する基盤。

**Why:** 非エンジニアが店舗別KPIをLINE WORKSで毎朝確認できるようにするため。対象は直営（company_id=1）のみ。

**How to apply:** 次の会話では「残り作業」の状態から引き継ぐこと。

---

## BigQuery 設定（確定済み）

| 項目 | 値 |
|------|----|
| project_id | `n8n-td-claudecode` |
| dataset | `test_n8n_td_claudecode` |
| location | `asia-northeast1` |
| 認証方式 | Service Account（credential ID: `zzJ2dnsMahTnqf38`） |

### 使用テーブル

| テーブル | 用途 |
|---------|------|
| `test_n8n_td_claudecode.test_l2_ficks_monitoring_kpi_join_day` | 日次・MTD累計 |
| `test_n8n_td_claudecode.test_l2_ficks_monitoring_kpi_join_month` | 月次確定値（来客周期の正確な計算用） |

**注意（2026-04-03確認）:**
- 月次テーブルはデータセットが `test_n8n_td_claudecode_monthly`（旧）→ `test_n8n_td_claudecode`（新）に移動
- テーブル名も `_monthly`（旧）→ `_month`（新）に変更済み
- 月次テーブルには `2026/04` データあり（310店）

### テーブル仕様の注意点
- 日次テーブル：同一 `shop_id × base_date` で複数行あり → `ROW_NUMBER() OVER (PARTITION BY shop_id ORDER BY time DESC)` で最新1行を取得
- `new_members` は NULL あり → `COALESCE(CAST(new_members AS INT64), 0)` で対処
- `ebitda` は全行 NULL のため使用不可
- `membership_count` は月次テーブルで全行0 → 会員数は日次テーブルから取得
- **直営フィルタ**: `company_id = 1`（それ以外は加盟店）
- **shop_group の値**: `既存`(110店) / `既存新店（FY24）`(25店) / `新店`(45店)
- **n8n SQLで `FORMAT_DATE('%Y-%m-%d', ...)` は使用禁止**: n8nが `%` を変数展開して0行になる不具合あり。代わりに `CAST(DATE_SUB(...) AS STRING)` / `CAST(DATE_TRUNC(..., MONTH) AS STRING)` を使うこと

---

## BQクエリ（確定版）

日次データ + 当月MTD累計を1クエリで取得。3行返却（既存 / 既存新店（FY24）/ 新店）。

```sql
WITH
daily AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY shop_id ORDER BY time DESC) as rn
  FROM `n8n-td-claudecode.test_n8n_td_claudecode.test_l2_ficks_monitoring_kpi_join_day`
  WHERE base_date = CAST(DATE_SUB(CURRENT_DATE('Asia/Tokyo'), INTERVAL 1 DAY) AS STRING)
    AND company_id = 1
),
mtd_raw AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY shop_id, base_date ORDER BY time DESC) as rn
  FROM `n8n-td-claudecode.test_n8n_td_claudecode.test_l2_ficks_monitoring_kpi_join_day`
  WHERE base_date >= CAST(DATE_TRUNC(CURRENT_DATE('Asia/Tokyo'), MONTH) AS STRING)
    AND base_date < CAST(CURRENT_DATE('Asia/Tokyo') AS STRING)
    AND company_id = 1
),
daily_agg AS (
  SELECT shop_group, COUNT(DISTINCT shop_id) as shop_count,
    ROUND(SUM(consumed_based_sales)) as d_sales,
    SUM(CAST(monthly_fee AS INT64)) as d_fee,
    SUM(CAST(membership_count AS INT64)) as d_members,
    SUM(new_visitors) as d_new_visitors,
    SUM(COALESCE(CAST(new_members AS INT64),0)) as d_new_members,
    SUM(CAST(membership_cancellations_count AS INT64)) as d_cancellations,
    SUM(CAST(membership_active_users_count AS INT64)) as d_active_users
  FROM daily WHERE rn = 1 GROUP BY shop_group
),
mtd_flow AS (
  SELECT shop_group,
    ROUND(SUM(consumed_based_sales)) as m_sales,
    SUM(new_visitors) as m_new_visitors,
    SUM(COALESCE(CAST(new_members AS INT64),0)) as m_new_members,
    SUM(CAST(membership_cancellations_count AS INT64)) as m_cancellations,
    SUM(total_visitors) as m_total_visitors,
    SUM(total_unique_visitors) as m_unique_visitors
  FROM mtd_raw WHERE rn = 1 GROUP BY shop_group
),
mtd_last AS (
  SELECT shop_group,
    SUM(CAST(monthly_fee AS INT64)) as m_fee,
    SUM(CAST(membership_count AS INT64)) as m_members,
    SUM(CAST(membership_active_users_count AS INT64)) as m_active_users
  FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY shop_id ORDER BY base_date DESC, time DESC) as rn2
    FROM mtd_raw WHERE rn = 1
  ) WHERE rn2 = 1
  GROUP BY shop_group
)
SELECT
  d.shop_group, d.shop_count,
  d.d_sales, d.d_fee, d.d_members,
  d.d_new_visitors, d.d_new_members,
  ROUND(SAFE_DIVIDE(d.d_new_members, NULLIF(d.d_new_visitors,0))*100, 1) as d_join_rate,
  d.d_cancellations,
  ROUND(SAFE_DIVIDE(d.d_cancellations, NULLIF(d.d_members,0))*100, 2) as d_cancel_rate,
  ROUND(SAFE_DIVIDE(d.d_active_users, NULLIF(d.d_members,0))*100, 1) as d_active_rate,
  f.m_sales,
  f.m_new_visitors,
  f.m_new_members,
  ROUND(SAFE_DIVIDE(f.m_new_members, NULLIF(f.m_new_visitors,0))*100, 1) as m_join_rate,
  f.m_cancellations,
  ROUND(SAFE_DIVIDE(f.m_cancellations, NULLIF(COALESCE(l.m_members, d.d_members),0))*100, 2) as m_cancel_rate,
  ROUND(SAFE_DIVIDE(f.m_total_visitors, NULLIF(f.m_unique_visitors,0)), 2) as m_visit_freq,
  l.m_fee,
  l.m_members,
  ROUND(SAFE_DIVIDE(l.m_active_users, NULLIF(l.m_members,0))*100, 1) as m_active_rate
FROM daily_agg d
LEFT JOIN mtd_flow f USING(shop_group)
LEFT JOIN mtd_last l USING(shop_group)
ORDER BY CASE d.shop_group WHEN '既存' THEN 1 WHEN '既存新店（FY24）' THEN 2 WHEN '新店' THEN 3 END
```

### 出力カラム一覧

| カラム | 意味 |
|--------|------|
| `d_sales` | 日次 消化売上 |
| `d_fee` | 日次 会員費合計 |
| `d_members` | 日次 会員数（スナップショット） |
| `d_new_visitors` | 日次 新規来客者数 |
| `d_new_members` | 日次 新規入会者数 |
| `d_join_rate` | 日次 新規入会率（%） |
| `d_cancellations` | 日次 退会者数 |
| `d_cancel_rate` | 日次 退会率（%） |
| `d_active_rate` | 日次 アクティブ率（%） |
| `m_sales` | MTD累計 消化売上（月初は NULL） |
| `m_fee` | MTD 会員費合計（月初は NULL） |
| `m_members` | MTD 会員数（月初は NULL） |
| `m_new_visitors` | MTD 新規来客者数（月初は NULL） |
| `m_new_members` | MTD 新規入会者数（月初は NULL） |
| `m_join_rate` | MTD 新規入会率（%） |
| `m_cancellations` | MTD 退会者数（月初は NULL） |
| `m_cancel_rate` | MTD 退会率（%） |
| `m_visit_freq` | MTD 来客周期（延べ来店÷ユニーク来客、月初は NULL） |
| `m_active_rate` | MTD アクティブ率（%） |

### 来客周期の定義
`月間延べ来店数 ÷ 月間ユニーク来客者数`（例: A+Bで6回来店→6÷2=3回）
- 日次は常に≒1.0になるため表示しない（`—`）
- MTDデータなし（月初）は NULL → `—` 表示

---

## n8n ワークフロー（確定版）

| 項目 | 値 |
|------|----|
| ワークフロー ID | `jhomsySSFmdWhzfH` |
| ワークフロー名 | `Daily Store Report → LINE WORKS` |
| ノード数 | 16 |
| 状態 | **Active（毎朝9時 Schedule有効）** |
| n8n URL | `http://10.0.2.10:5678` |
| n8n APIキー | `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJiMDMwYmJkZi05MzliLTRhMDEtOWE0OS0wM2E0YThhMzllYTMiLCJpc3MiOiJuOG4iLCJhdWQiOiJwdWJsaWMtYXBpIiwianRpIjoiY2ZmMmI0YzAtYWM4YS00Nzg2LThkMWYtMzMyMjhhZjAwYmFmIiwiaWF0IjoxNzc1MDI5ODUyLCJleHAiOjE3Nzc2MDgwMDB9.xAXq6SVqp0BSsPP33DTupvZAmSTe5j2QcCyiSAxlJfk` |

### フロー構成

```
[手動実行（テスト用）] または [毎朝9時実行（Schedule）]
 ↓
[BQ: KPI日次データ取得]（Google BigQuery）
  → 3行返却（既存 / 既存新店（FY24）/ 新店）
 ↓
[Flex Message 生成]（Code）
 ↓ （4並列）
 ├─→ [Merge: データ+Token] ← [Get line token]
 │     ↓
 │   [Merge: 全データ合流] ← [Daily Report チャネルID]
 │     ↓
 │   [リクエストボディ生成]
 │     ↓
 │   [LINE WORKS POST] → [完了]
 │
 └─→ [宛先リスト] ──────────→ [Merge: メールデータ]
     [メールHTML生成] ────────→ [Merge: メールデータ]
                                       ↓
                                  [Gmail送信]
```

### LINE WORKS 通知仕様（Carousel 3枚）

各分類（既存・既存新店・新店）ごとにカード1枚。
カード色：既存＝青(`#1a73e8`)、既存新店＝緑(`#0f9d58`)、新店＝オレンジ(`#e65100`)

各カードに以下を表示（日次 ＋ MTD累計 の2列）：
- 消化売上 / 会員費 / 会員数
- 新規来客者数 / 新規入会者数 / 新規入会率
- 退会者数 / 退会率
- 来客周期（MTDのみ）/ アクティブ率

**LINE WORKSのFlex Message制約（ハマりポイント）:**
- `rgba()` 形式の色コードは非対応 → `#rrggbb` のみ使用可
- `spacing: 'none'` は非対応 → 使用禁止
- Carousel最大10枚

### Gmail 通知仕様
- 宛先: `stanley.ng@nobitel.co.jp`, `k-yuki@nobitel.co.jp`, `t-goto2@nobitel.co.jp`
- 件名: `📊 KPI実績レポート YYYY-MM-DD`
- 本文: HTML形式（既存/既存新店/新店の3テーブル、日次・MTD累計の2列）

### 表示ルール
- NULL・0値は `—` 表示（`fmtSales`/`fmtNum`/`fmtRate`/`fmtFreq` 関数で制御）
- 会員費・会員数はMTDデータあればMTD、なければ日次値を表示
- 月初（当月データ0日分）は MTD列が全て `—`（正常動作）

---

## 動作確認状況

| 日付 | 実行ID | 状況 |
|------|--------|------|
| 2026-04-01 | 5825 | ✓ Gmail送信成功、LINE WORKS 201 |
| 2026-04-02 | 5833 | ✓ 両方成功（BQ 4/1データあり） |
| 2026-04-03 | 5851 | BQ 0行（FORMAT_DATE % 問題）→ CAST書き換え済み |

---

## 関連リソース

| 項目 | 値 |
|------|----|
| LINE WORKS Bot ID | `1420096` |
| line_token DataTable ID | `6iipvD4qo8LCSyrZ` |
| Daily Report チャネルID | `a462b64a-d9ec-f929-ed0a-908920e22445` |
| Google BigQuery credential | `zzJ2dnsMahTnqf38` |
| Google Sheets credential | `stAsixak35mlitqr` |
| Bot API エンドポイント | `https://www.worksapis.com/v1.0/bots/{botId}/channels/{channelId}/messages` |
| Sub-Workflow Line通知（流用元） | `SZyfRwL0wwTII6F5` |

---

## 残り作業

| 項目 | 状態 |
|------|------|
| 4/3以降のSchedule実行で正常動作確認（CAST書き換え後） | 未確認 |
| TD → BQ Export 設定 | 未完了 |
| 予算・昨期比カラムの追加 | 未着手（budget_33期/34期_monthlyテーブルあり、要設計） |
