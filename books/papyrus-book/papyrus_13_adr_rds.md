---
title: "ADR: RDS は当面 Single-AZ とする"
---

### 背景（Context）

* 検証用ポートフォリオ。常時可用は要件外。コスト最小を優先。

### 決定（Decision）

* RDS Single-AZ。自動バックアップ＋PITR有効化。月次で**実復旧ドリル**を実施。

### 代替案（Alternatives）

* 障害時はRTO 30–60分/RPO 数分で復旧。ダウンタイムは許容。

### 影響（Consequences）

* 読める証跡を最小で残すメリット

### 記録（Decision Record）

* `docs/evidence/<ts>_rds_restore.log`, `<ts>_snapshots.json`, Alarms定義（`infra/30-monitor`）。

### フォローアップ（Follow-ups）

* トラフィック/MTTRが閾値超過で `multi_az=true` に移行。移行Runbookあり。

短答：**減点ではない。**「可用性の要件に対して妥当な根拠と復旧手順が固定化されているか」を見られる。Multi-AZは“常時可用”のための追加コスト。**RTO/RPOを宣言して、手順と証跡を出せるならシングルAZでも評価は落ちない。**

## 逆に突っ込まれがちなポイントと返し

* Q: 「障害時、誰がどの順で何をするの？」
  A: 「Runbookに従い、CloudWatch Alarm発火→オーナーが`restore-db-instance-to-point-in-time`実行→エンドポイント切替→/healthz→/dbcheck で検証→完了。**証跡はPR化**します。」

* Q: 「Multi-AZじゃないのに“高可用”って言ってない？」
  A: 「言っていません。“**復旧しやすい構成**”として説明しています。**常時可用が要件化したらMulti-AZ**に上げます。」

* Q: 「復旧時間は本当にその範囲で終わる？」
  A: 「**実測の証跡**を貼ります。サイズが増えた場合の見積もりもRunbookに書いてあります。」

まとめると、**“意図と代替策を先に言う”**のがコツ。
「安くしてる理由」と「壊れた時の手順」と「後で上げられる設計」を**固定化＋証跡**で示せば、Multi-AZじゃなくても普通に加点圏内。

---

## 面接での言い方（そのまま使えるやつ）

* 「用途はポートフォリオ検証環境で、**ダウンタイム許容**です。**コスト最小**を優先し、RDSは**Single-AZ**運用にしています。」
* 「代わりに**RTO/RPOを明示**しました。RTOは＜X分〜Y分＞、RPOは＜数分以内＞を目標に、**自動バックアップ＋PITR**と**スナップショット復旧のランブック**で担保しています。」
* 「**復旧手順は固定化**済みで、`docs/evidence` に**実際に復旧した証跡**を置いています。将来トラフィックが増えたら `multi_az=true` 切り替えの**IaCタスク**を用意しています。」

> これで「安いけど準備はしてる」評価に寄せられる。

---

## “コストカット + 復旧手段確立”を主張するなら、必ず添えるもの

1. **SLO（暫定）を1行で**

   * 例: 「運用RTO 30–60分、RPO 数分（PITR想定）」

2. **Runbook（要約）**

   * `Restore to point in time → new DB identifier → 置換/DSN切替 → 健康確認 → 古い方を退役`
   * これを**固定手順**として README/ADR に載せる

3. **証跡**

   * 自動バックアップ有効のスクショ/CLI出力
   * 1回は**実復旧テスト**を実施して `*_restore.log` を `docs/evidence` に保存
   * CloudWatch Alarms（接続失敗/CPU/ストレージ）を `infra/30-monitor` に定義

4. **将来の切り替え計画**

   * Terraformの `multi_az`（または**Multi-AZ DB instance / Multi-AZ cluster**への移行）を**別ブランチ**で用意
   * 想定ダウンタイムと手順をADRに明記

---


