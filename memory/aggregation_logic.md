---
name: BQクエリ 集計ロジック詳細
description: 日次+MTD累計クエリのCTE設計、flow/stock分離、NULL処理、n8n固有の制約
type: project
---

## 集計の全体設計

BQクエリ1本で「昨日の日次値」と「当月MTD累計」を同時に取得する。
返却行数: **3行**（shop_group別 = 既存 / 既存新店（FY24）/ 新店）

---

## CTE 構造

```
daily      → 昨日の全店データ（重複排除）
mtd_raw    → 当月1日〜昨日の全店全日データ（重複排除）
daily_agg  → daily を shop_group 別に SUM・COUNT
mtd_flow   → mtd_raw を shop_group 別に累計SUM（flow系指標）
mtd_last   → mtd_raw の各店舗の最終日スナップショットを shop_group 別に SUM（stock系指標）
```

最後に `daily_agg LEFT JOIN mtd_flow LEFT JOIN mtd_last` で結合。

---

## daily CTE（昨日の日次データ）

```sql
SELECT *, ROW_NUMBER() OVER (PARTITION BY shop_id ORDER BY time DESC) as rn
FROM `...test_l2_ficks_monitoring_kpi_join_day`
WHERE base_date = CAST(DATE_SUB(CURRENT_DATE('Asia/Tokyo'), INTERVAL 1 DAY) AS STRING)
  AND company_id = 1
```

- `ROW_NUMBER() PARTITION BY shop_id ORDER BY time DESC` で同一shop_id×base_dateの重複を除去（最新1行）
- `rn = 1` でフィルタ
- **直営のみ**: `company_id = 1`（加盟店は除外）

## mtd_raw CTE（当月MTD生データ）

```sql
SELECT *, ROW_NUMBER() OVER (PARTITION BY shop_id, base_date ORDER BY time DESC) as rn
FROM `...test_l2_ficks_monitoring_kpi_join_day`
WHERE base_date >= CAST(DATE_TRUNC(CURRENT_DATE('Asia/Tokyo'), MONTH) AS STRING)
  AND base_date < CAST(CURRENT_DATE('Asia/Tokyo') AS STRING)
  AND company_id = 1
```

- 対象期間: **当月1日 ≦ base_date < 今日**（昨日まで）
- 月初（今日が月の1日）はこの条件が `0日分` → 0行 → MTD列全部 NULL → `—` 表示（正常）
- `PARTITION BY shop_id, base_date` で日ごと・店ごとの重複を除去

---

## mtd_flow CTE（累計SUM系指標）

**対象**: 日ごとに加算していくべき指標

| カラム | 集計方法 |
|--------|---------|
| `m_sales` | `ROUND(SUM(consumed_based_sales))` ← **正式カラム名（確定）** |
| `m_new_visitors` | `SUM(new_visitors)` |
| `m_new_members` | `SUM(COALESCE(CAST(new_members AS INT64), 0))` |
| `m_cancellations` | `SUM(CAST(membership_cancellations_count AS INT64))` |
| `m_total_visitors` | `SUM(total_visitors)` ← 来客周期の分子 |
| `m_unique_visitors` | `SUM(total_unique_visitors)` ← 来客周期の分母 |

**なぜ flow を別CTEにするか**: 会員数・会員費・アクティブ率は「最終日の値」を使うべき（累計するとおかしくなる）。flow系だけ SUM で集計する。

---

## mtd_last CTE（最終日スナップショット系指標）

**対象**: ある日のスナップショットを使うべき指標（累計してはいけない）

```sql
SELECT shop_group,
  SUM(CAST(monthly_fee AS INT64)) as m_fee,
  SUM(CAST(membership_count AS INT64)) as m_members,
  SUM(CAST(membership_active_users_count AS INT64)) as m_active_users
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY shop_id ORDER BY base_date DESC, time DESC) as rn2
  FROM mtd_raw WHERE rn = 1
) WHERE rn2 = 1
GROUP BY shop_group
```

- 内側サブクエリで `PARTITION BY shop_id ORDER BY base_date DESC, time DESC` → 各店舗の **最終日最新行** を取得（rn2=1）
- その値を shop_group 別に SUM

| カラム | 意味 |
|--------|------|
| `m_fee` | MTD期間内の最終日時点の会員費合計 |
| `m_members` | MTD期間内の最終日時点の会員数合計 |
| `m_active_users` | MTD期間内の最終日時点のアクティブ会員数合計 |

---

## 全KPI項目の計算ロジック

画像（LINE WORKS / メール通知）に表示される全項目の定義。

### 消化売上
- **定義**: 当日（または当月MTD）に会員が実際に利用した売上。請求ベースではなく消化ベース。
- **BQカラム**: `consumed_based_sales` ← **正式カラム名（ユーザー確認済み・確定）**
- **重要**: **ストレッチ＋指名料金の合計（税抜き）**。
  - 含む: ストレッチ料金 ＋ 指名料金
  - 税抜き金額
  - 検証: 二子玉川店（shop_id: 2051）4/4実績 → 指名¥18,000 + ストレッチ¥195,143 = **¥213,143**（BQ値と一致確認済み）
- **集計**: flow系・累計SUM
- 日次: `ROUND(SUM(consumed_based_sales))`
- MTD: `ROUND(SUM(consumed_based_sales))`

### 会員費
- **定義**: 店舗が保有する会員から徴収する月額会員費の合計。日によって変動しないストック値。
- **BQカラム**: `monthly_fee`
- **集計**: stock系・最終日スナップショット
- 日次: `SUM(CAST(monthly_fee AS INT64))`
- MTD: mtd_last の `m_fee`（MTD期間内の各店舗最終日の値をSUM）
- 日次=MTD になることが多い（最終日=昨日=日次の値と一致するため正常）

### 会員数
- **定義**: 店舗が保有するアクティブ会員の総数。累計ではなく「その時点の在籍数」。
- **BQカラム**: `membership_count`
- **集計**: stock系・最終日スナップショット
- 日次: `SUM(CAST(membership_count AS INT64))`
- MTD: mtd_last の `m_members`
- 注意: 月次テーブルでは全行0になるバグあり → 必ず日次テーブルから取得

### 新規来客者数
- **定義**: その日（または当月MTD）に初めて来店した人数。リピーターは含まない。
- **BQカラム**: `new_visitors`
- **集計**: flow系・累計SUM
- 日次: `SUM(new_visitors)`
- MTD: `SUM(new_visitors)`

### 新規入会者数
- **定義**: その日（または当月MTD）に新たに会員登録した人数。
- **BQカラム**: `new_members`（NULLあり）
- **集計**: flow系・累計SUM
- 日次: `SUM(COALESCE(CAST(new_members AS INT64), 0))`
- MTD: `SUM(COALESCE(CAST(new_members AS INT64), 0))`

### 新規入会率
- **定義**: 新規来客者のうち、その場で入会した割合。来店→入会への転換率。
- **計算式**: 新規入会者数 ÷ 新規来客者数 × 100
- 日次: `ROUND(d_new_members / d_new_visitors * 100, 1)`
- MTD: `ROUND(m_new_members / m_new_visitors * 100, 1)`

### 退会者数
- **定義**: その日（または当月MTD）に退会した会員の人数。
- **BQカラム**: `membership_cancellations_count`
- **集計**: flow系・累計SUM
- 日次: `SUM(CAST(membership_cancellations_count AS INT64))`
- MTD: `SUM(CAST(membership_cancellations_count AS INT64))`

### 退会率
- **定義**: 保有会員数に対して退会した割合。会員の離脱率を示す。
- **計算式**: 退会者数 ÷ 会員数 × 100
- 日次: `ROUND(d_cancellations / d_members * 100, 2)`
- MTD: `ROUND(m_cancellations / m_members * 100, 2)`
- 分母の会員数はstock系（最終日スナップショット）を使用

### 来客周期
- **定義**: 月間で1人の会員が平均何回来店したか。延べ来店数÷ユニーク来客者数。
- **計算式**: 月間延べ来店数 ÷ 月間ユニーク来客者数
- **BQカラム**: `total_visitors`（延べ）/ `total_unique_visitors`（ユニーク）
- **集計**: flow系・MTDのみ表示
- 日次: `—` 固定（日次では常に≒1.0になるため意味がない）
- MTD: `ROUND(SAFE_DIVIDE(m_total_visitors, NULLIF(m_unique_visitors, 0)), 2)`
- 例: 2人が月6回来店 → 6÷2=3.0回

### アクティブ率
- **定義**: 保有会員のうち、実際に来店・利用しているアクティブ会員の割合。
- **BQカラム**: `membership_active_users_count`
- **集計**: stock系・最終日スナップショット
- 日次: `ROUND(d_active_users / d_members * 100, 1)`
- MTD: mtd_last の `ROUND(m_active_users / m_members * 100, 1)`
- 注意: 画像では日次=`—`、MTD=`—` の場合あり（データ欠損時）

### 日次の注意点
- 日次の会員費・会員数・アクティブ率は「その日の最新スナップショット」
- 日次の来客周期は `—` 固定（分母のユニーク来客者が累計でないと意味をなさないため）
- 日次のアクティブ率は「昨日時点の値」として表示

### MTDの注意点
- 会員費・会員数・アクティブ率: mtd_last（MTD期間内の最終日スナップショット）を使用
- 消化売上・新規来客・新規入会・退会者数: mtd_flow（累計SUM）を使用
- 画像例: 会員費の日次=MTD=¥8369万 → 同じ値になるのは正常（MTD最終日=昨日 = 日次の値と一致）
- 画像例: 会員数の日次=MTD=27,493名 → 同上

---

## 来客周期の計算

**定義**: 月間延べ来店数 ÷ 月間ユニーク来客者数

```sql
ROUND(SAFE_DIVIDE(f.m_total_visitors, NULLIF(f.m_unique_visitors, 0)), 2) as m_visit_freq
```

- `m_total_visitors` = MTD期間の全来店数（同じ人の複数回来店を含む）
- `m_unique_visitors` = MTD期間のユニーク来客者数
- 例: A+Bの2人が合計6回来店 → 6÷2=3.0回
- **日次は常に ≒ 1.0 になるため表示しない**（`—` 固定）
- MTDデータなし（月初）は NULL → `—` 表示

---

## NULL・0値の処理ルール

| 状況 | 処理 |
|------|------|
| `new_members` に NULL あり | `COALESCE(CAST(new_members AS INT64), 0)` |
| 分母が0になる割り算 | `SAFE_DIVIDE(分子, NULLIF(分母, 0))` |
| 月初（MTDデータ0日分） | mtd_flow・mtd_lastが0行 → LEFT JOIN で NULL → 表示側で `—` |
| `ebitda` は全行NULL | 使用禁止 |
| `membership_count` 月次テーブルで全行0 | 会員数は日次テーブルから取得すること |

---

## 表示側のフォールバックルール

- 会員費・会員数: `m_fee`/`m_members` があればMTD値、なければ日次値 `d_fee`/`d_members` を表示
- アクティブ率: `m_active_rate` があればMTD、なければ日次 `d_active_rate`
- 来客周期: MTDデータなし → `—`（フォールバック値を計算しない）

---

## n8n BQノード 固有の制約

### FORMAT_DATE 禁止

```sql
-- NG: n8nが%を変数展開して0行になる
WHERE base_date = FORMAT_DATE('%Y-%m-%d', DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))

-- OK: CASTを使う
WHERE base_date = CAST(DATE_SUB(CURRENT_DATE('Asia/Tokyo'), INTERVAL 1 DAY) AS STRING)
WHERE base_date >= CAST(DATE_TRUNC(CURRENT_DATE('Asia/Tokyo'), MONTH) AS STRING)
```

n8nのCode/BQノードはSQL内の `%Y`, `%m`, `%d` を変数プレースホルダとして解釈してしまい、空文字に展開されてWHERE条件が壊れる。`CAST(...AS STRING)` を使うことで回避できる。

---

## shop_group の値と店舗数（2026-04時点）

| shop_group | 店舗数 |
|------------|--------|
| 既存 | 110店 |
| 既存新店（FY24） | 25店 |
| 新店 | 45店 |

ORDER BY: 既存=1, 既存新店（FY24）=2, 新店=3

---

## 月初動作の期待値

| 項目 | 動作 |
|------|------|
| 日次指標（d_*） | 通常通り取得（昨日 = 前月最終日） |
| MTD指標（m_*） | 全て NULL → `—` 表示（正常） |
| 来客周期（m_visit_freq） | NULL → `—` 表示（正常） |
