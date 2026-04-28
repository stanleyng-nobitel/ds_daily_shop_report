---
name: 34期・33期グループマッピング（確定版）
description: 34期と33期のshop_groupの対応関係。BQビューのprev_day CTE・yoy CTE・result_33_daily参照時に必須。
type: project
originSessionId: 023b5a16-1f16-4340-9c37-e6a68924ec6e
---
## 34期（現在）↔ 33期（前年）のグループ対応（確定）

| 34期 shop_group | 33期 group_33 / store_type |
|---|---|
| 既存（135店） | 既存 ＋ 既存新店（合算） |
| 既存新店（18店） | 新店（FY25） |
| 新店（29店） | 対応なし（前年NULLになる） |

**Why:**
- 33期「既存新店」として存在した店舗が34期では「既存」に昇格
- 33期「新店（FY25）」として存在した店舗が34期では「既存新店」に昇格
- 34期「新店」29店はまだオープンしていない（budget_34_dailyにも存在しない）
- budget_34_daily の group_34 値は「既存」「既存新店」のみ（新店なし）

**How to apply:**

### prev_day CTE（result_33_dailyのgroup_33 → 34期shop_group）
```sql
CASE group_33
  WHEN '既存新店'    THEN '既存'      -- 33期 既存新店 → 34期 既存に統合
  WHEN '新店（FY25）' THEN '既存新店' -- 33期 新店（FY25）→ 34期 既存新店に昇格
  ELSE group_33                        -- '既存' → '既存'
END AS shop_group
```
- 既存の d_prev_sales = 33期 既存 ＋ 既存新店 の合計
- 既存新店の d_prev_sales = 33期 新店（FY25）の合計
- 新店の d_prev_sales = NULL（前年比較なし）

### yoy CTE（budget_33_monthlyのstore_type → 34期shop_group）
```sql
CASE store_type
  WHEN '既存'        THEN '既存'
  WHEN '既存新店'    THEN '既存'      -- 34期 既存に統合（GROUP BY 1 必須）
  WHEN '新店（FY25）' THEN '既存新店' -- 34期 既存新店に昇格
END AS shop_group
-- ※ GROUP BY store_type ではなく GROUP BY 1 にすること（集計重複防止）
```

### n8nメールHTML生成コードのGROUP設定
```javascript
const GROUP_LABELS = {'既存':'既存','既存新店':'既存新店','新店':'新店'};
const GROUP_COLORS = {'既存':'#1a73e8','既存新店':'#0f9d58','新店':'#e65100'};
const GROUP_ORDER  = ['既存','既存新店','新店'];
// FY24サフィックスは不要（実データのshop_groupは「既存新店」）
```
