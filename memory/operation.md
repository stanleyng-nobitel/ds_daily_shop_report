---
name: 運用・データフロー設計の整理
description: TD→BQ→n8nのデータフロー、MTDロジックの整合性確認、BQビュー構成、メールレポート設計
type: project
---

## TDワークフロー設定（確定版）

```yaml
timezone: Asia/Tokyo
schedule:
  daily>: 04:30:00

_export:
  td:
    database: l2_ficks
  date_from: "${moment(session_date).subtract(1, 'month').format('YYYY-MM-DD')}"
  date_to:   "${moment(session_date).subtract(1, 'day').format('YYYY-MM-DD')}"
  split: "day"
```

**処理ステップ:**
1. `+delete_old_record` — `monitoring_kpi_join_day` を毎回DELETE（重複排除）
2. `+date_trunc_list` — 日付リストをループして日次集計（`p210_data_aggregate_middle_range_day`）
3. `+insert_to_bigquery` — **日次処理が全部終わった後に** BQへINSERT（TD job ID: 2863341）

---

## データフロー全体像

```
毎日 04:30  TD ワークフロー開始
            ① monitoring_kpi_join_day を DELETE
            ② 日付ループで日次集計（約30日分）
            ③ 全日次処理完了後 → BQへ INSERT
     〜午前中  処理完了（4時間程度）

     06:00  n8n: budget_34_daily 更新（Ms6M1fOCGvTQYWid）
            → Google Sheetsの日次予算シートからBQへ書き込み

     12:00  n8n: Daily Store Report（jhomsySSFmdWhzfH）
            → BQからKPIデータ取得 → LINE WORKS / Gmail 通知
```

**注意**: `+insert_to_bigquery` は日次ループが全部終わった後に実行されるため、BQへの反映は04:30+処理時間後になる。

---

## BQテーブル・ビュー構成（確定版）

### ソーステーブル（TDから流入）

| テーブル | 内容 | 粒度 |
|---------|------|------|
| `result_34_daily_l2_ficks_monitoring_kpi_join_day` | 34期 日次実績（店舗×日） | 店舗×日 |
| `result_34_monthly_l2_ficks_monitoring_kpi_join_month` | 34期 月次MTD実績 | 店舗×月 |

### 手動管理テーブル（GSheet→n8n→BQ: ワークフロー `4EMcCjVPj2voPEtd`）

| テーブル | 内容 | スキーマ |
|---------|------|---------|
| `budget_34_daily` | 34期 店舗別日次予算 | shop_id, kpi_category, budget_amount, budget_date |
| `budget_34_monthly` | 34期 グループ別月次予算（全月合計） | group_34, group_33, kpi_name, unit, month, value |
| `result_33_monthly` | 33期 グループ別月次実績 | group_34, group_33, kpi_name, unit, month, value |
| `result_33_daily` | 33期 店舗別日次実績 | shop_id等, date, stretch_sales |

### BQビュー

| ビュー | 用途 | 参照テーブル |
|-------|------|------------|
| `v_daily_kpi_with_budget` | メールレポート用グループ集計 | result_34_daily, result_34_monthly, budget_34_daily, budget_34_monthly, result_33_monthly, result_33_daily |
| `v_dashboard_shop_daily` | ダッシュボード用店舗別日次 | result_34_daily, budget_34_daily |

---

## v_daily_kpi_with_budget の CTE構成（確定版）

| CTE | 内容 |
|-----|------|
| `daily` | 昨日の店舗別実績（rn=1で重複排除） |
| `db` | 昨日の店舗別日次予算（budget_34_daily） |
| `mb_grp` | 今月のグループ別月次予算（**budget_34_monthly**から取得 ← 月の全日分合計） |
| `dow_avg` | 曜日別平均（着地計算用、直近35日） |
| `remaining_by_dow` | 残り日の曜日別カウント |
| `forecast` | 着地見込み（実績 + 残り日曜日平均） |
| `daily_agg` | グループ別日次集計 |
| `monthly_agg` | グループ別月次MTD集計（result_34_monthly） |
| `yoy` | 昨年同月のグループ別実績（result_33_monthly、同月全月合計） |
| `prev_day` | 昨年同日の日次実績（result_33_daily） |

**重要**: 月次予算は `budget_34_monthly` の当月全日合計を使用（`budget_34_daily` のMTD合計ではない）。

---

## メールレポート（jhomsySSFmdWhzfH）の構成

### 表示項目（グループ別テーブル：既存・既存新店）

| 項目 | 日次予算 | 日次実績 | 達成率 | 月次累計予算 | 月次累計実績 | 着地（見込） | 予算比（見込）|
|-----|---------|---------|-------|------------|------------|------------|-------------|
| 消化売上(税抜) | ✓ | ✓（前年同日付） | ✓ | ✓ | ✓（**前年同月**） | ✓ | ✓ |
| 会員費 | — | — | — | — | ✓（**前年同月**） | — | — |
| 会員数 | — | — | — | — | ✓（**前年同月**） | — | — |
| 新規来客者数 | ✓ | ✓ | ✓ | ✓ | ✓（**前年同月**） | ✓ | ✓ |
| 新規入会者数 | ✓ | ✓ | ✓ | ✓ | ✓（**前年同月**） | ✓ | ✓ |
| 新規入会率 | — | ✓ | — | **✓（予算入会率）** | ✓（**前年同月**） | — | — |
| 退会者数 | — | — | — | — | ✓（**前年同月**） | ✓ | — |
| 退会率 | — | — | — | — | ✓ | — | — |
| 来客周期 | — | — | — | — | ✓ | — | — |
| アクティブ率 | — | — | — | — | ✓ | — | — |

**前年同月**: `yoy_*` 列（result_33_monthlyの昨年同月全月合計）を紫色小テキストで表示
**予算入会率**: JS内で `m_budget_new_members / m_budget_new_visitors * 100` を計算

---

## TDのデータ取得ウィンドウ

- **ローリングウィンドウ**: 常に「約30日前〜昨日」のデータを取得
- **BQへの書き込み方式**: DELETE → INSERT（毎日全件入れ替え）

例: 今日が4/7 → BQには 3/7〜4/6 のデータが入る

---

## MTDロジックとの整合性

### BQ側のMTDフィルター
```sql
WHERE base_date >= CAST(DATE_TRUNC(CURRENT_DATE('Asia/Tokyo'), MONTH) AS STRING)
  AND base_date < CAST(CURRENT_DATE('Asia/Tokyo') AS STRING)
```
→ 当月1日〜昨日を集計

### 問題ないか？ → 問題なし

| 今日 | TDのwindow | MTD範囲 | BQに含まれる？ |
|------|-----------|---------|--------------|
| 4/7 | 3/7〜4/6 | 4/1〜4/6 | ✓ |
| 5/31 | 4/30〜5/30 | 5/1〜5/30 | ✓ |
| 6/1 | 5/1〜5/31 | 0日分 → `—` | ✓（月初正常動作） |
| 6/2 | 5/2〜6/1 | 6/1〜6/1 | ✓ |

---

## 過去に発生した問題と原因

### 4/4・4/5 shop_group崩れ

**原因**: n8nが9時に実行した時点で、shop_group分けSQL（12時スタート）がまだBQに反映されていなかった。

**対処**: BQ取得前にデータ品質チェックノードを追加。`COUNT(DISTINCT shop_group) != 3` の場合はLINE WORKSにアラートを送信してKPI通知をスキップ。

---

## n8n スケジュール設定

| ワークフロー | スケジュール | 内容 |
|------------|------------|------|
| `jhomsySSFmdWhzfH` | 毎日 12:00 | Daily Store Report → LINE WORKS + Gmail |
| `Ms6M1fOCGvTQYWid` | 毎日 06:00 | 日次予算 GSheet → budget_34_daily |
| `4EMcCjVPj2voPEtd` | 手動実行（月次更新時） | result33_budget34_month GSheet → result_33_monthly + budget_34_monthly |

---

## BQテーブルの書き込み仕様まとめ

| テーブル | 書き込み方式 | 更新タイミング |
|---------|------------|--------------|
| result_34_daily（TD経由） | DELETE→INSERT | 04:30〜午前中 |
| budget_34_daily | TRUNCATE→INSERT (n8n) | 毎日 06:00 |
| budget_34_monthly | TRUNCATE→Wait10分→INSERT | 手動（月次更新時）|
| result_33_monthly | TRUNCATE→Wait10分→INSERT | 手動（月次更新時）|

**budget_34_daily修正履歴（2026-04-14）**:
- `FORMATTED_VALUE` → `UNFORMATTED_VALUE`（生の値を取得）
- `Math.round(amount * 100) / 100` で小数点2桁固定
- 新規客数・入会者数は0.2〜0.8人台の小数値があるため丸め禁止

---

## v_dashboard_shop_daily のデータ確認（2026-04-17時点）

- 182店舗 × 16日（4/1〜4/16）= 2,912行 ✓
- 既存新店の4店舗（イオン名古屋茶屋/多摩平の森/岡崎/洛北阪急）は `b_sales=0` だが意図的（新規来客予算はあり）
- メールKPIとダッシュボードの月次実績差異: 新規入会者数で1%未満の誤差あり（集計元テーブルが異なるため正常）
