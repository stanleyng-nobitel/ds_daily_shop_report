---
name: KPI計算定義（周期・アクティブ率）
description: 周期とアクティブ率の計算式とBQカラム名
type: project
---

## 周期
**定義:** 会員制リピーターが1人あたり月に何回来店したか

**計算式:**
```
周期 = customer_count_repeater_membership / unique_customer_count_repeater_membership
```

**BQカラム:**
- 分子: `customer_count_repeater_membership`（客数_リピーター_会員制）
- 分母: `unique_customer_count_repeater_membership`（客数_ユニーク_リピーター_会員制）

---

## アクティブ率
**定義:** 会員のうち実際に来店したユニーク人数の割合

**計算式:**
```
アクティブ率 = unique_customer_count_repeater_membership / membership_count × 100
```

**BQカラム:**
- 分子: `unique_customer_count_repeater_membership`（客数_ユニーク_リピーター_会員制）
- 分母: `membership_count`（会員数）

---

**Why:** ダッシュボードに追加予定のKPI指標。GSheet定義ファイルで確認済み。

**How to apply:** HTMLダッシュボードのKPIS配列に追加する際はこの計算式を使う。BQクエリでは `customer_count_repeater_membership / NULLIF(unique_customer_count_repeater_membership, 0)` の形で安全に計算する。
