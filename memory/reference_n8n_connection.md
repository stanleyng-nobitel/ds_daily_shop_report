---
name: n8n接続情報
description: n8nインスタンスのURL・APIキー・主要ワークフローID・手動実行方法
type: reference
originSessionId: 023b5a16-1f16-4340-9c37-e6a68924ec6e
---
**URL:** `http://localhost:5678`（内部）/ `http://10.0.2.10:5678`（UI）

**API Key:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJiMDMwYmJkZi05MzliLTRhMDEtOWE0OS0wM2E0YThhMzllYTMiLCJpc3MiOiJuOG4iLCJhdWQiOiJwdWJsaWMtYXBpIiwianRpIjoiZDVlMWU1N2UtZGQ2Yy00NTM2LWE2MDktMjI2ODlhY2FlYzAzIiwiaWF0IjoxNzczMzE0MDY0fQ.MBLtKfCZfsDI0b_4ucL48NR_0rIMUwWf4WdHoHiznfM
```

**APIキーの場所:** `/Users/n8n/.claude/settings.json` → `mcpServers.n8n-mcp.env.N8N_API_KEY`

---

## 主要ワークフロー

| ID | 名前 | スケジュール | 内容 |
|----|-----|------------|------|
| `jhomsySSFmdWhzfH` | Daily Store Report → LINE WORKS | 毎日 12:00 | BQ v_daily_kpi_with_budget → メールHTML + LINE WORKS Flex通知 |
| `Ms6M1fOCGvTQYWid` | Budget Sheet to BQ budget_34_daily | 毎日 06:00 | GSheet日次予算シート → budget_34_daily |
| `4EMcCjVPj2voPEtd` | result33_budget34_month → BQ | 手動実行 | GSheet「result33_budget34_month」タブ → result_33_monthly + budget_34_monthly |

---

## 手動実行方法（Scheduleトリガーワークフロー）

ScheduleトリガーのワークフローはREST APIで直接実行不可。以下の手順で実行：

```python
# 1. Webhookノードを追加してPUT /api/v1/workflows/{id}
webhook_node = {
    'id': 'temp-webhook-node-001',
    'name': '手動実行Webhook',
    'type': 'n8n-nodes-base.webhook',
    'typeVersion': 2,
    'position': [-672, 1160],
    'disabled': False,
    'parameters': {
        'httpMethod': 'GET',
        'path': 'manual-run-daily-report',
        'responseMode': 'onReceived',
        'options': {}
    },
    'webhookId': 'manual-run-daily-report-001'
}
# connections['手動実行Webhook'] = {'main': [[{'node': '最初のノード名', 'type': 'main', 'index': 0}]]}

# 2. Webhookを叩く
# curl -s "http://localhost:5678/webhook/manual-run-daily-report"

# 3. 完了後Webhookノードを削除してPUT /api/v1/workflows/{id}
```

**注意**: ワークフロー更新時は `n8n_update_full_workflow` または REST API PUT を使う。
`n8n_update_partial_workflow` の `updateNode` でBQノードを更新するとcredentialsが消える。

---

## 認証情報（credential ID）

| 用途 | Credential ID |
|-----|--------------|
| BigQuery サービスアカウント | `zzJ2dnsMahTnqf38` |
| LINE WORKS Bot トークン | `6iipvD4qo8LCSyrZ` |
| Google Sheets OAuth2 | `stAsixak35mlitqr` |

---

## n8nワークフロー更新時の注意点

- **BQノードを含む場合は必ず `n8n_update_full_workflow` を使う**（partial updateでcredentials消失）
- Mergeノードは `mode: "append"` を使う（`combinationMode: "multiplex"` はエラーになる）
- BQ expression内の `/` は `\/` にエンコードされる問題あり → SQLはCodeノードで文字列生成してから渡す
