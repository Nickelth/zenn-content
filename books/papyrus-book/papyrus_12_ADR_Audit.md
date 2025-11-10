---
title: "ADR: 観測の入口（1行JSON＋HTTP生ログ＋監査分離）"
---

## 監査要件の意思決定

### Context

* スモークと監査の利害相反（ノイズ/遅延/機微）

### Decision

* CloudWatch 1行JSON＋HTTP生ログをCIで収集／監査は別WF

### Alternatives（Fluent系/外部APM 等）

* 比較対象と不採用理由を簡潔に

### Consequences（ノイズ・権限・可読性）

* 読める証跡を最小で残すメリット

### Decision Record

* YYYY-MM-DD, Owner

### Follow-ups（可観測性の拡張案）

* sslrootcert検証／p90→SLO設計／分散トレース導入検討