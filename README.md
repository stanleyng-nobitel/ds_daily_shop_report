# ds_daily_shop_report

BigQuery に蓄積された店舗KPIデータを n8n で毎朝集計し、LINE WORKS（Flex Message）と Gmail（HTML）に自動通知する基盤。

### 33期予算（日次）「作期実績（日次） 」：
https://docs.google.com/spreadsheets/d/14YhcFvjToD7esslEjQrTb4a12NKz67t9-9Avg96wRT8/edit?gid=319497420#gid=319497420

### 33期実績（月次）「作期実績＆予算（原本）」：
https://docs.google.com/spreadsheets/d/14YhcFvjToD7esslEjQrTb4a12NKz67t9-9Avg96wRT8/edit?gid=638680827#gid=638680827

ーーー

### 34期予算（日次）「予算（日次）」：
https://docs.google.com/spreadsheets/d/14YhcFvjToD7esslEjQrTb4a12NKz67t9-9Avg96wRT8/edit?gid=0#gid=0

### 34期予算（月次）「作期実績＆予算（原本）」：
https://docs.google.com/spreadsheets/d/14YhcFvjToD7esslEjQrTb4a12NKz67t9-9Avg96wRT8/edit?gid=638680827#gid=638680827

### その他（UIテンプレート）
https://docs.google.com/spreadsheets/d/1HosRJaT9mmcxrcB9OpEn3uvN-gtup5m6u_9cCWxOBd8/edit?gid=0#gid=0

### TD店舗更新（新店→既存新店→既存）
p200_/02_shop_group_master/sql

https://docs.google.com/spreadsheets/d/1Efu8KUTEyf5x-BZMg0OVleHcBe_irv-2Z8OtGEhT4nQ/edit?gid=0#gid=0

---

## 概要

| 項目 | 内容 |
|------|------|
| 通知先 | LINE WORKS（Carousel 3枚）+ Gmail |
| 通知タイミング | 毎日昼 12:00（n8n Schedule） |
| 集計対象 | 直営店（company_id=1）のみ |
| 分類 | 既存 / 既存新店 / 新店 の3グループ |
| データソース | BigQuery（project: `n8n-td-claudecode`、dataset: `test_n8n_td_claudecode`） |

---

## BQテーブル構成

dataset: `n8n-td-claudecode.test_n8n_td_claudecode`

### テーブル一覧

| テーブル名 | 種別 | 更新 | 内容 |
|---|---|---|---|
| `result_34_daily_l2_ficks_monitoring_kpi_join_day` | TABLE | TD 毎日4:30 | 34期日次KPI実績（Fixcs）。直近約31日ローリング。レポートの主データソース |
| `result_34_monthly_l2_ficks_monitoring_kpi_join_month` | TABLE | TD 毎日10:30 | 34期月次KPI実績（Fixcs）。2024-04以降の全月次データ |
| `budget_34_daily` | TABLE | n8n 毎日6:00 | 34期日次予算。予算（日次）シートから取得。TRUNCATE→全件INSERT |
| `budget_34_monthly` | TABLE | 手動（一回限り） | 34期月次予算。作期実績＆予算（原本）シートから取得 |
| `result_33_daily` | TABLE | n8n 毎日6:00 | 33期日次実績。作期実績（日次）シートから取得。TRUNCATE→全件INSERT |
| `result_33_monthly` | TABLE | 手動（一回限り） | 33期月次実績（確定値）。作期実績＆予算（原本）シートから取得 |
| `v_daily_kpi_with_budget` | VIEW | — | レポート用中間集計ビュー。shop_group単位で日次・MTD・予算・月末予測を集計 |
| `v_dashboard_shop_daily` | VIEW | — | 店舗別日次ダッシュボード用。直近8日分の実績+予算を店舗単位で出力 |
| `v_report1_shop_sales` | VIEW | — | 店舗別売上内訳（支払/消化/現金/チケット） |
| `v_report2_member_analysis` | VIEW | — | 会員数・入退会・来店回数分布 |
| `v_report3_stretch_operations` | VIEW | — | 顧客タイプ別施術時間・ベッド稼働率 |
| `v_report4_area_brand` | VIEW | — | エリア・ブランド別集計・ベッド当たり指標 |
| `v_report5_acquisition_conversion` | VIEW | — | 新規獲得・会員/チケット転換率 |

### v_daily_kpi_with_budget（ビュー詳細）

レポートの直接データソース。`shop_group` 単位で以下を1行に横持ち出力する。

**参照テーブル**
- `result_34_daily_l2_ficks_monitoring_kpi_join_day` — 前日実績・当月MTD・直近35日（予測用）
- `budget_34_daily` — 前日予算・当月MTD予算・月合計予算

**出力カラムのプレフィックス**

| プレフィックス | 意味 |
|---|---|
| `d_` | 前日の日次実績 |
| `d_budget_` | 前日の日次予算 |
| `m_` | 当月MTD累計実績（月初〜前日） |
| `m_budget_` | 当月MTD累計予算 |
| `mb_total_` | 月合計予算（月末までの全日合計） |
| `fc_` | 月末着地予測（直近35日の曜日別平均 × 残り日数） |

**CTE構成**

| CTE | 内容 |
|---|---|
| `daily` | 前日実績（`time`降順で重複除去） |
| `mtd_raw` | 当月1日〜前日のMTD実績（同上） |
| `db` | 前日の予算（3KPI） |
| `mb_daily` | 当月MTD予算（日次積上げ） |
| `mb_monthly_grp` | 月合計予算（グループ別） |
| `dow_avg` | 直近35日の曜日別平均 |
| `remaining_by_dow` | 今日〜月末の残り曜日カウント |
| `forecast` | 月末着地予測（`dow_avg × remaining_by_dow`） |
| `daily_agg` | グループ別日次集計（実績+予算） |
| `mtd_flow` | グループ別MTD集計（実績+予算） |
| `mtd_cancel` | グループ別MTD退会者数 |
| `yoy` | 前年同月実績（`result_33_monthly`から取得） |
| `prev_day` | 前日の33期実績（`result_33_daily`から取得） |

### その他ビュー（v_dashboard / v_report1〜5）

全て `result_34_daily_l2_ficks_monitoring_kpi_join_day` を参照。フィルタなしで全データを展開。

| ビュー | 主な用途 |
|---|---|
| `v_dashboard_shop_daily` | 直近8日分の店舗別実績+予算（`a_` = 実績、`b_` = 予算） |
| `v_report1_shop_sales` | 売上内訳。`total_cash_sales` / `total_ticket_sales` の計算カラムあり |
| `v_report2_member_analysis` | 会員分析。来店回数分布（`*_visits_0〜ge_5_monthly`）含む |
| `v_report3_stretch_operations` | 施術オペレーション。`bed_utilization_rate`（ベッド稼働率）計算あり |
| `v_report4_area_brand` | エリア・ブランド横断。`sales_per_bed` / `visitors_per_bed` 計算あり |
| `v_report5_acquisition_conversion` | 新規転換率。`new_to_membership_rate` / `new_to_3ticket_rate` / `trial_to_3ticket_rate` |

### データフロー

```
Fixcs（店舗SaaS）
  └─ TD _z_p200_data_aggregate_middle_branch_daily.dig（毎日4:30）
       └─ result_34_daily_l2_ficks_monitoring_kpi_join_day
  └─ TD 【本番】n8n-td-claudecode_month（毎日10:30）
       └─ result_34_monthly_l2_ficks_monitoring_kpi_join_month

Google Sheets「予算（日次）」
  └─ n8n Ms6M1fOCGvTQYWid（毎日6:00）
       └─ budget_34_daily

result_34_daily_l2... + budget_34_daily
  └─ v_daily_kpi_with_budget（BQビュー）
       └─ n8n jhomsySSFmdWhzfH（毎日12:00）
            ├─ LINE WORKS（Flex Message）
            └─ Gmail（HTML）
```

---

## ファイル構成

```
ds_daily_shop_report/
├── .env               # APIキー・IDなどの機密情報（git管理外）
├── .gitignore
├── README.md
└── memory/            # Claude Code のメモリファイル（プロジェクト記録）
    ├── MEMORY.md            # メモリインデックス
    ├── project_daily_report.md  # ワークフロー・BQ設定・残り作業
    ├── aggregation_logic.md     # BQクエリの集計ロジック詳細
    └── user_profile.md          # ユーザープロファイル
```

---

## セットアップ

### 1. .env を作成する

`.env.example` を参考に `.env` を作成してください。

```bash
cp .env.example .env
```

設定が必要な変数：

| 変数名 | 説明 |
|--------|------|
| `N8N_API_KEY` | n8n の API キー |
| `LINEWORKS_BOT_ID` | LINE WORKS Bot ID |
| `LINEWORKS_CHANNEL_ID` | 通知先チャネル ID |
| `LINEWORKS_BOT_API_ENDPOINT` | Bot API エンドポイント URL |
| `N8N_LINE_TOKEN_DATATABLE_ID` | n8n DataTable（LINE トークン管理）ID |
| `N8N_SUB_WORKFLOW_LINE_ID` | LINE 通知サブワークフロー ID |
| `GCP_BIGQUERY_CREDENTIAL_ID` | n8n 内の BigQuery credential ID |
| `GCP_SHEETS_CREDENTIAL_ID` | n8n 内の Google Sheets credential ID |

### 2. n8n ワークフローの確認

- ワークフロー ID: `jhomsySSFmdWhzfH`
- ワークフロー名: `Daily Store Report → LINE WORKS`
- n8n URL: `http://10.0.2.10:5678`
- ワークフローが **Active** になっていることを確認する

---

## 手動実行（テスト）

n8n の管理画面からワークフロー `Daily Store Report → LINE WORKS` を開き、「Execute Workflow」ボタンで手動実行できます。

正常時の動作：
1. BigQuery から昨日の日次 + 当月MTD累計を取得（3行）
2. LINE WORKS に Flex Message（Carousel）を送信
3. Gmail に HTML メールを送信

---

## 主要KPI

通知に含まれる指標：

| 指標 | 日次 | MTD |
|------|------|-----|
| 消化売上 | ✓ | ✓ |
| 会員費 | ✓ | ✓ |
| 会員数 | ✓ | ✓ |
| 新規来客者数 | ✓ | ✓ |
| 新規入会者数 | ✓ | ✓ |
| 新規入会率 | ✓ | ✓ |
| 退会者数 | ✓ | ✓ |
| 退会率 | ✓ | ✓ |
| 来客周期 | — | ✓ |
| アクティブ率 | ✓ | ✓ |

集計ロジックの詳細は [memory/aggregation_logic.md](memory/aggregation_logic.md) を参照。

---

## 注意事項

- BQ クエリで `FORMAT_DATE('%Y-%m-%d', ...)` は使用禁止（n8n が `%` を変数展開して0行になる）
  - 代わりに `CAST(DATE_SUB(...) AS STRING)` を使うこと
- LINE WORKS の Flex Message で `rgba()` と `spacing: 'none'` は非対応
- 月初はMTD列が全て `—` になる（正常動作）
