---
title: "ADR: 観測の入口（1行JSON＋HTTP生ログ＋監査分離）"
---

## 監査要件の意思決定

### 背景（Context）

* スモークと監査の利害相反（ノイズ/遅延/機微）

### 決定（Decision）

* CloudWatch 1行JSON＋HTTP生ログをCIで収集／監査は別WF

### 代替案（Alternatives）

* 比較対象と不採用理由を簡潔に

### 影響（Consequences）

* 読める証跡を最小で残すメリット

### 記録（Decision Record）

* YYYY-MM-DD, Owner, 適用範囲

### フォローアップ（Follow-ups）

* sslrootcert検証／p90→SLO設計／分散トレース導入検討