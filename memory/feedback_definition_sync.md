---
name: KPI定義変更時はn8nも必ず同期
description: KPI計算定義を変更した場合、n8nワークフローも必ず同時に更新する
type: feedback
---

KPI定義（周期・アクティブ率など）をドキュメントやダッシュボードで更新したら、n8nワークフロー `jhomsySSFmdWhzfH`（Daily Store Report → LINE WORKS）のCodeノードも必ず同じ定義に合わせて更新すること。

**Why:** LINE WORKS通知とHTMLダッシュボードで異なる定義を使うと、数値が食い違い混乱を招く。一貫性を保つため定義は一元管理する。

**How to apply:** 定義変更の作業が完了したら「n8nも更新しましたか？」と自分でチェックする。更新対象ノードは「Flex Message 生成」と「メールHTML生成」の2つ。
