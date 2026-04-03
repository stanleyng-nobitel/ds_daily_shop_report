# ds_daily_shop_report

BigQuery に蓄積された店舗KPIデータを n8n で毎朝集計し、LINE WORKS（Flex Message）と Gmail（HTML）に自動通知する基盤。

---

## 概要

| 項目 | 内容 |
|------|------|
| 通知先 | LINE WORKS（Carousel 3枚）+ Gmail |
| 通知タイミング | 毎朝 9:00（n8n Schedule） |
| 集計対象 | 直営店（company_id=1）のみ |
| 分類 | 既存 / 既存新店（FY24）/ 新店 の3グループ |
| データソース | BigQuery（project: `n8n-td-claudecode`） |

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
