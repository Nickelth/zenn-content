---
title: "コスト最適化の副作用：PrismaClientInitializationError の原因は“夜間停止×夜間ジョブ”"
emoji: "⌨"
type: "idea"
topics: ["aws", "prisma", "rds"]
published: false
---

※実務で得た知見を個人環境で再現したもので構成しています。

## 抄録

### 0. 要約

背景: 個人AWSで従量課金を抑えるため、RDS を深夜停止。

事象: 2025年4月から深夜に `PrismaClientInitializationError` が散発。

原因: 定期バッチテストが深夜実行のまま。DB 停止時間と衝突。

決定: バッチを「DB 到達不可ならスキップ」に設計変更し、実行時刻も RDS 起動後へ寄せる。

結果: 夜間の失敗 100% → 0%。無駄なアラート/失敗ログが消滅。

教訓: コスト最適化は SLO とスケジュールの三点セットで考える。

次の一手: “スキップ件数”をメトリクス化、月次で閾値超えたら見直す。

### 1. なぜ失敗したか（構造）

コスト最適化で「RDS を止める」という意思決定をしたのに、ジョブの実行窓はそのまま。設計上の整合性ミス。

### 2. 再現と証拠の取り方

CloudWatch Logs Insights で発生日の分布を出す

```sql
fields @timestamp, @message
| filter @message like /PrismaClientInitializationError/
| stats count() by bin(1d)
```

### 3. 対策（壊さずに直す）

スケジュール整合: バッチを RDS 起動後にずらす。JST 06:10 にしたいなら EventBridge は UTC 前提なので `cron(10 21 * * ? *)`。

ヘルスゲート: 実行前に「DB につながるか」をチェック。つながらなければ失敗ではなくスキップで終了。

バックオフ: 一時的な起動中をやり過ごすため指数バックオフ＋ジッタを最大数回。


### 4. 結果

| 指標 | 修正前 | 修正後 | 備考 |
| --- | --- | --- | --- |
| 深夜帯失敗率  100% |  0% | 4月以降のログ集計 |
| 夜間アラート件数/月 | N | 0 | 通知疲れ解消 |
| 実行所要時間中央値  | X分 | X分 | 変化なし（設計のみ変更）|

### 5. 考察

失敗を“ゼロにする”より、失敗しても壊さない設計に寄せたのが現実解。スキップの定義と観測可能性が肝。

### 6. 妥当性の脅威

期間が短い、他要因の混入、休日スケジュールのズレなどは今後の監視で補う。

### スクショ計画（証拠は軽く重く）

- 4月から夜間に失敗が集中している棒グラフ（1枚）。
- 修正後1週間の“失敗ゼロ/スキップゼロ”の棒グラフ（1枚）。
- EventBridge のスケジュール画面は差分だけ赤枠で（1枚）。

### この記事で“学位っぽく”見せるコツ

- 「原因はRDS停止中の実行スケジュール」と言い切る。
- 失敗を“撲滅”ではなく“スキップ可能にして無害化”と表現。
- 反実仮想は無理にやらない。代わりに「スキップ件数を監視し続ける」運用設計で締める。
- ズッコケは素材として最高。恥は一度だけ、教訓は永久保存。これを“手順”ではなく“決定ログ”で出せば、読む側はあなたの頭と手の両方を信用する。

```ts
// run-job.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();
const maxRetries = 3;

const sleep = (ms: number) => new Promise(r => setTimeout(r, ms));
const jitter = (n: number) => Math.floor(Math.random() * n);

async function canConnect(): Promise<boolean> {
  try {
    await prisma.$connect();
    await prisma.$disconnect();
    return true;
  } catch {
    return false;
  }
}

async function main() {
  for (let i = 0; i < maxRetries; i++) {
    if (await canConnect()) {
      // ここで本体処理を実行
      process.exit(0);
    }
    const backoff = Math.min(30000, 1000 * 2 ** i) + jitter(500);
    await sleep(backoff);
  }
  // ここまで来たらDB到達不可。失敗ではなく「スキップ」で正常終了扱いにする。
  // 観測のために "skipped_due_to_db_down" をログ出力。
  console.log('skipped_due_to_db_down');
  process.exit(0);
}

main().catch(() => process.exit(1));
```