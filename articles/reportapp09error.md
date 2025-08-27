---
title: "AWS構築 | 全体アーキテクチャ、SLO、コスト戦略の総まとめ"
emoji: "💥"
type: "tech"
topics: ["githubactions", "yaml", "cicd" ]
published: false
---

## ADR: <決定名>

- 状況: <前提/制約/対象>
- 選択肢: <A/B/C>（比較軸: 可用性/運用/コスト/セキュリティ）
- 決定: <採用案>
- 根拠: <採用理由3点>
- 反証/却下理由: <採用しなかった案の弱点>
- リスクと対策: <検知/回復/フォールバック>
- 影響: <影響範囲/トレードオフ>

### 見出しテンプレ

- 背景と前提（非機能要件・制約）
- 設計の決定（ADRミニ：選択肢/決定/却下理由/リスク）
- 設定抜粋（JSON/YAMLは芯だけ、ダミー化。全文は付録）
- 検証と証拠（plan/ドライラン、イベント履歴、メトリクス）
- 運用シナリオ（正常系、失敗時、切り戻し、所要時間）
- コストとセキュリティの影響
- 学びと代替案（やらなかったこと・次にやること）

### 証跡

- ポリシー抜粋とdiff（Action/Resource/Conditionの芯）
- Access Analyzer or Configの是正前後1枚
- CloudTrailでロール更新の記録

## エラー対処集

### 0. はじめに

### 1. エラー


### 7. 関連記事

#### AWS 関連

@[card](https://zenn.dev/nickelth/articles/reportapp044cicd)
@[card](https://zenn.dev/nickelth/articles/reportapp043ecs)
@[card](https://zenn.dev/nickelth/articles/reportapp042aws)
@[card](https://zenn.dev/nickelth/articles/reportapp041iam)