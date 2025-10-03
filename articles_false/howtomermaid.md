---
title: "Mermaidクイックスタートガイドとサンプルコード"
emoji: "💬"
type: "tech"
topics: ["markdown", "mermaid", "VSCode", "npm", "node"]
published: false
---

## Mermaid 使い方の超要約

* 3バッククオートでコードブロック、言語は `mermaid`。
* 最初の行が図の種類（`flowchart` とか `sequenceDiagram`）。
* 方向は `flowchart TD` みたいに指定（TB/TD/LR/RL/BT）。
* オンラインの Mermaid Live Editor で即実行。 ([mermaid.js.org][1])

### 0) ローカルクイックスタート

0. (前提) `npm`と`Node.js`をインストール。(無理ならブラウザへ。[mermaid.js.org][1])

1. VSCodeで拡張機能を3つ導入。

https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid
https://marketplace.visualstudio.com/items?itemName=bpruitt-goddard.mermaid-markdown-syntax-highlighting
https://marketplace.visualstudio.com/items?itemName=shd101wyy.markdown-preview-enhanced

2. ファイルの拡張子を`.md`にする。

3. ファイルをVSCodeで開き、 Ctrl+Shift+Vでダイアグラムがプレビューされる。

---

### 1) フローチャート（flowchart）

ノードと矢印。迷ったらまずこれ。

```mermaid
flowchart TD
  A[開始] --> B{条件?}
  B -- はい --> C[処理1]
  B -- いいえ --> D[処理2]
  C --> E[終了]
  D -. 注釈的な点線 .-> E
  subgraph 外部システム
    X[(DB)]
  end
  C -->|SQL| X
```

* 角括弧で形変わる：`[長方形]`、`(丸)`、`{ひし形}`、`((円))`、`[(シリンダー=DB)]` など。
* サブグラフは `subgraph ... end`。
* エッジラベルは `-- 文言 -->`。 ([docs.mermaidchart.com][2])

---

### 2) シーケンス図（sequenceDiagram）

やり取りの順番を時系列で殴る。

```mermaid
sequenceDiagram
  actor U as User
  participant A as App
  participant S as Service
  U->>A: ログイン要求
  A->>S: 認証API
  S-->>A: 200 OK + token
  A-->>U: ログイン成功
  rect rgba(0,0,0,0.05)
    note over A,S: リトライ制御
    A->>S: リトライ
  end
  alt トークン失効
    A->>S: refresh
  else 有効
    A-->>U: 画面表示
  end
```

* `actor/participant`、`alt/else/end`、`rect`、`note`、`loop` などが使える。
* `end` という単語は一部環境で衝突するので必要なら括弧などで囲え、という注意がある。 ([docs.mermaidchart.com][3])

---

### 3) クラス図（classDiagram）

関係性の骨組みを素早く書く用。

```mermaid
classDiagram
  class User {
    +id: int
    +name: string
    +login(): bool
  }
  class Admin {
    +ban(user: User)
  }
  Admin --|> User
  User "1" --> "*" Session : owns
```

* 可視性 `+/-/#`、関連 `"1" -- "*"`, 継承 `--|>`、実装 `..|>` などが基本。 ([mermaid.js.org][4])

---

### 4) 状態遷移図（stateDiagram）

状態機械を殴る時に。

```mermaid
stateDiagram
  [*] --> Idle
  Idle --> Working : start
  Working --> Error : fail
  Working --> Idle : done
  Error --> Idle : reset
```

* 初期 `[∗]`、終端 `[∗]`。サブ状態も書ける。 ([mermaid.js.org][4])

---

### 5) ER 図（erDiagram）

データモデルを口頭会議で殴り倒す時用。

```mermaid
erDiagram
  USER ||--o{ ORDER : places
  ORDER ||--|{ ORDER_LINE : contains
  USER {
    int id
    string name
  }
  ORDER {
    int id
    date created_at
  }
```

* 関連の多重度 `||--o{` など。テーブル定義風の属性記述ができる。 ([mermaid.js.org][4])

---

### 6) ガントチャート（gantt）

期限の可視化。上司の「いつ終わるの」攻撃を防御。

```mermaid
gantt
  title 10月の作業計画
  dateFormat  YYYY-MM-DD
  section API
  設計           :a1, 2025-10-01, 2d
  実装           :a2, after a1, 4d
  section インフラ
  IaC整備        :b1, 2025-10-03, 5d
```

* `dateFormat`、`after` 参照が便利。 ([mermaid.js.org][4])

---

### 7) ユーザージャーニー（journey）

体験フローと感情の起伏。

```mermaid
journey
  title ログイン体験
  section 既存ユーザー
    フォーム表示: 3: ユーザー
    2FA入力: 2: ユーザー
    ダッシュボード表示: 4: ユーザー
```

* 数字は満足度。プレゼン用の即席図に良い。 ([mermaid.js.org][4])

---

### 8) 円グラフ（pie）

数字の内訳を雑に見せる時。

```mermaid
pie showData
  title エラー原因
  "設定ミス" : 55
  "仕様誤解" : 30
  "不可抗力" : 15
```

([mermaid.js.org][4])

---

### 9) マインドマップ（mindmap）

ブレストの下書きに最速。

```mermaid
mindmap
  root((採用向けポートフォリオ))
    アプリ
      ECS/ECR
      CloudWatch
    ドキュメント
      ADR
      構成図
```

([mermaid.js.org][4])

---

### 10) タイムライン（timeline）

イベントの並びだけ出したい時。

```mermaid
timeline
  title 進捗
  2025-10-01 : 設計開始
  2025-10-05 : MVP完成
  2025-10-10 : 公開
```

([mermaid.js.org][4])

---

### 11) Git グラフ（gitGraph）

ブランチ戦国時代を可視化。

```mermaid
gitGraph
  commit id: "init"
  branch feature
  checkout feature
  commit
  checkout main
  merge feature
```

([mermaid.js.org][4])

---

### 12) 要件図・クアドラントなど（requirementDiagram / quadrantChart ほか）

* 要件図は `requirementDiagram`、優先度と検証手段を整理するのに使える。
* クアドラントは `quadrantChart`。2軸マトリックスを一撃で描ける。
* ただし対応状況はレンダラやバージョン依存あり。使う前に公式の「Diagram Syntax」一覧で生存確認してから突撃。C4やSankey等は注意書き付き。 ([mermaid.js.org][4])

---

### よく使う小技

```mermaid
%% コメント
%%{init: {'theme': 'forest'}}%%  %% エディタや埋め込みで初期設定
```

* `%%{init: ...}%%` でテーマや設定を上書き。
* ノードやエッジのスタイル変更は `style id fill:#eee,stroke:#333` みたいに書く。細かいスタイルは SVG 寄りの指定。 ([mermaid.js.org][5])

---

### 公式サイトと Live Editor

まずここで動けばだいたい勝ち。社内 Wiki 連携や拡張（VS Code 拡張など）もあるが、まずはブラウザで確実に通す。 ([mermaid.js.org][1])

---

## 参考（公式ドキュメント）

* Diagram Syntax（総合リファレンス）と各ページ（Flowchart, Sequence, Class…）を叩け。 ([mermaid.js.org][5])

[1]: https://mermaid.live/edit#pako:eNpVjcFOg0AQhl9lMydNaENbCLAHE0u1lyZ66EnoYQIDSyy7ZFlSK_DuLjTGOqeZfN__Tw-Zygk4FGd1yQRqw467VDI7z0ksdNWaGtsTWyyehj0ZVitJ14FtH_aKtUI1TSXLx5u_nSQW94dJI2ZEJT_HG4rn_Jukge2SAzZGNad7cryogb0k1buw9f-J0GRTr0mBvMBFhprFqGcFHCh1lQM3uiMHatI1Tif0E03BCKopBW7XnArsziaFVI421qD8UKr-TWrVlQJs_bm1V9fkaGhXYanxTyGZk45VJw1wf24A3sMX8M0mXPrBOopCz_V91w8DB67WcZdrL4oCb7XaeJElowPf8093GQa-ezer8QfIaHaP "Mermaid | Diagramming and charting tool"
[2]: https://docs.mermaidchart.com/mermaid-oss/syntax/flowchart.html "Flowcharts – Basic Syntax"
[3]: https://docs.mermaidchart.com/mermaid-oss/syntax/sequenceDiagram.html "Sequence diagrams | Mermaid"
[4]: https://mermaid.js.org/syntax/classDiagram.html "Class diagrams"
[5]: https://mermaid.js.org/intro/syntax-reference.html "Diagram Syntax"
