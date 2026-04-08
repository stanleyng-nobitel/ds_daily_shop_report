---
name: LINE WORKS チャットbot 開発プラン
description: LINEから自然言語でBQをクエリして返すbot。Google Sheets→n8n→Claude API（MCP）→BQのアーキテクチャ
type: project
---

# Plan: LINE WORKS チャットbot → Google Sheets → n8n → MCP Server（Claude + BQ tool）

## Context
LINEチャンネルにメッセージを送ると自然言語でBQをクエリして結果を返すbotを作る。
- READ専用（SELECT のみ）
- Prompt Injection / SQL Injection 防止
- **n8n の MCP Server ノードを使って Claude API を呼び出し、BQツールを Claude が直接実行**
- 外部公開不要: Google Sheetsをバッファとして使用

---

## Google Sheets 構造（確認済み）

シートID: `1-W7qEjd3TTpy_zVbnZG2jALzDNdzu41vBpEJTq8aHr0` / GID: `511132926`

| カラム | 内容 |
|--------|------|
| A: `loggedAt` | ログ記録日時 |
| B: `createdAt` | メッセージ作成日時 |
| C: `userId` | 送信者ユーザーID |
| D: `accountId` | アカウントID |
| **E: `channelId`** | 返信先チャンネルID |
| F: `messageType` | メッセージ種別 |
| **G: `text`** | メッセージ本文 |
| H: `UserName` | 送信者名 |
| I: `processed` | **追加**: 空=未処理 / `done`=処理済み |
| J: `response` | **追加**: 返答ログ |

**セキュリティ制約**: `channelId = a462b64a-d9ec-f929-ed0a-908920e22445` のみBQ実行許可

---

## アーキテクチャ

```
LINE WORKS メッセージ送信
  ↓（既存GAS Webhook）
Google Sheets（メッセージバッファ）
  ↑ ポーリング（n8nのSchedule）
n8n
  ↓ channelId チェック（a462b64a-... のみ）
  ↓ Prompt Injectionチェック（Code）
  ↓ n8n MCP Server ノード（Claude API呼び出し）
      Claude: メッセージを解析してBQクエリを生成・実行
      Claude に渡すtools: { bigquery_query: SELECT実行のみ }
  ↓ 結果フォーマット（Code）
  ↓ Get line token（DataTable）
  ↓ LINE WORKS 返信（HTTP Request）
  ↓ Google Sheets: processed=done に更新
```

---

## 実装詳細

### Step 1: Google Sheetsにカラム追加

列I: `processed`、列J: `response` を追加

### Step 2: n8n 新規ワークフロー（9ノード）

| # | ノード名 | 種類 | 役割 |
|---|---------|------|------|
| 1 | ポーリング | Schedule | 毎分（n8n最短）またはWebhookトリガー |
| 2 | Sheets: 未処理取得 | Google Sheets | processed=空の行を取得 |
| 3 | チャンネルID確認 | IF | `a462b64a-...` のみ通過 |
| 4 | Prompt Injectionチェック | Code | 危険キーワード・500文字超を拒否 |
| 5 | **MCP Server: Claude + BQ** | MCP Client | Claude APIを呼び、BQツールでSELECT実行 |
| 6 | 結果フォーマット | Code | テキスト整形 |
| 7 | Get line token | DataTable | 既存流用（`6iipvD4qo8LCSyrZ`） |
| 8 | LINE WORKS 返信 | HTTP Request | Bot ID `1420096` / channelId はSheetから |
| 9 | Sheets: 処理済み更新 | Google Sheets | processed=done, response=返答 |

### Step 3: MCP Server ノード設定

n8n の **@n8n/n8n-nodes-mcp** または **HTTP Request → Claude API** で実装。

**Claudeに渡すSystem Prompt:**
```
あなたはBigQuery店舗KPIデータの照会アシスタントです。
SELECTクエリのみ実行できます。

利用可能なテーブル:
`n8n-td-claudecode.test_n8n_td_claudecode.test_l2_ficks_monitoring_kpi_join_day`

必須ルール:
- SELECT のみ（INSERT/UPDATE/DELETE/DROP等は絶対禁止）
- WHERE company_id = 1 を必ず含める
- LIMIT 20 を必ず付与
- 日付: CAST(DATE_SUB(CURRENT_DATE('Asia/Tokyo'), INTERVAL 1 DAY) AS STRING) = 昨日

カラム:
shop_name(店舗名), shop_id, shop_group(既存/既存新店（FY24）/新店),
base_date(日付STRING), consumed_based_sales(消化売上・税抜・ストレッチ+指名含む),
monthly_fee(会員費), membership_count(会員数),
new_visitors(新規来客者数), new_members(新規入会者数),
membership_cancellations_count(退会者数),
total_visitors(延べ来店数), total_unique_visitors(ユニーク来客数),
membership_active_users_count(アクティブ会員数)

返答は日本語で簡潔に。数値は見やすく整形すること。
```

**Claudeに渡すtool定義（BQ実行）:**
```json
{
  "name": "bigquery_query",
  "description": "BigQueryでSELECTクエリを実行する",
  "input_schema": {
    "type": "object",
    "properties": {
      "sql": { "type": "string", "description": "実行するSELECT文" }
    },
    "required": ["sql"]
  }
}
```

n8nがtool_useを受け取ったらBQ BigQueryノードで実行し、結果をtool_resultとして返す。

### Step 4: Prompt Injectionチェック（Codeノード）

```javascript
const msg = ($input.first().json.text || '').trim();
const channelId = $input.first().json.channelId;
const rowIndex = $input.first().json.row_number;

if (msg.length > 500) {
  return [{ json: { blocked: true, reason: '文字数超過', channelId, rowIndex } }];
}
const forbidden = ['DROP','DELETE','UPDATE','INSERT','TRUNCATE','ALTER',
  'CREATE','GRANT','EXEC','--','/*','UNION','INFORMATION_SCHEMA'];
if (forbidden.some(k => msg.toUpperCase().includes(k))) {
  return [{ json: { blocked: true, reason: '禁止キーワード', channelId, rowIndex } }];
}
return [{ json: { blocked: false, message: msg, channelId, rowIndex } }];
```

---

## セキュリティ設計（4重防護）

| レイヤー | 対策 |
|---------|------|
| ① チャンネルID制限 | Daily Reportチャンネルのみ処理 |
| ② Prompt Injection防止 | 500文字超・危険キーワードで入力を拒否 |
| ③ Claude System Prompt | SELECTのみ・LIMIT 20・company_id=1を明示的に強制 |
| ④ n8n tool実行前検証 | tool_useのSQL内容をn8n Codeノードで再チェックしてから実行 |

---

## 変更・作成ファイル

- **新規**: n8n ワークフロー（既存 `jhomsySSFmdWhzfH` とは別）
- **修正**: Google Sheets に `processed`（I列）・`response`（J列）追加
- **参照**: `.env`（BQ credential: `zzJ2dnsMahTnqf38`、line token: `6iipvD4qo8LCSyrZ`、Sheets credential: `stAsixak35mlitqr`）
- **更新**: `memory/project_daily_report.md`

---

## 検証手順

1. Google Sheetsに `processed`・`response` カラムを追加
2. LINEで「渋谷店の昨日のデータを見せて」と送信
   → Sheetsにメッセージが記録され、n8nが処理してLINE返信が届くことを確認
3. LINEで「既存グループの売上TOP5」と送信 → ランキングが返ることを確認
4. LINEで「DELETE FROM test」と送信 → エラーメッセージが返ることを確認
5. Google Sheetsの `processed=done` と `response` に返答が記録されることを確認
