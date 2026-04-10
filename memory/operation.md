---
name: 運用・データフロー設計の整理
description: TD→BQ→n8nのデータフロー、MTDロジックの整合性確認、運用上の注意点
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

     12:00  n8n Schedule 実行
            → BQからKPIデータ取得 → LINE WORKS / Gmail 通知
```

**注意**: `+insert_to_bigquery` は日次ループが全部終わった後に実行されるため、BQへの反映は04:30+処理時間後になる。9時時点で反映済みかどうかは処理量による。

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
| 6/30 | 5/30〜6/29 | 6/1〜6/29 | ✓ |

**理由**: MTDに必要なのは「当月1日以降」のデータ。TDのrolling windowは常に約30日分あるため、当月1日は必ずwindow内に収まる。

---

## 過去に発生した問題と原因

### 4/4・4/5 shop_group崩れ

**原因**: n8nが9時に実行した時点で、shop_group分けSQL（12時スタート）がまだBQに反映されていなかった。全店が`新店`として入っている中間状態のデータを取得してしまった。

**対処**: BQ取得前にデータ品質チェックノードを追加。`COUNT(DISTINCT shop_group) != 3` の場合はLINE WORKSにアラートを送信してKPI通知をスキップ。

```
[BQ: データ品質チェック] → [IF: shop_group 3種類？]
  ├─ true  → 通常のKPI通知フロー
  └─ false → LINE WORKS アラート送信（「TreasureData側のデータ取得を確認してください」）
```

**恒久対策の候補**:
- n8nのScheduleを9時 → 13時以降に変更（shop_group分けSQL完了後）
- または品質チェックのアラートを受けて手動再実行する運用を継続

---

## n8n スケジュール設定

| 項目 | 値 |
|------|-----|
| 現在のSchedule | 毎日昼 12:00（`0 12 * * *`） |
| shop_group SQL完了時刻 | 12:00以降（推定） |
| 変更履歴 | 9:00 → 12:00 に変更済み（2026-04-08） |

---

## BQテーブルの書き込み仕様まとめ

| 項目 | 値 |
|------|-----|
| 書き込み方式 | TRUNCATE（毎日全件入れ替え） |
| 対象テーブル | `test_n8n_td_claudecode.test_l2_ficks_monitoring_kpi_join_day` |
| データ保持期間 | 約30日分（TDのrolling window） |
| 更新タイミング | 04:30開始 → 処理時間約4時間 → 午前中に完了 |
| TDジョブID（BQ INSERT） | 2863341 |
| TDジョブID（DELETE） | 2869872 |
| TDデータベース | l2_ficks |
