---
title: "降りかかる説明責任をgit blameコマンドで回避する"
emoji: "💨"
type: "tech"
topics: ["git", "cli", "bash"]
published: false
---

※実務で得た知見を個人環境で再現したもので構成しています。

## git blame, git logコマンドの役割とそのバリエーション

### 目的の再定義
炎上ワードは表現を着替える。目的は「変更責任の可視化と監査耐性」。

### 集める証拠

ライン単位の“最終更新者”出力スクリプト例と結果

--ignore-rev-file適用前後の差

CODEOWNERSやレビューフローの導入差分

### 見出しの骨格

なぜ可視化が必要か（監査/事故調）

blameの限界と改善策（整形コミット無視、範囲特定）

ツール化（出力フォーマット、運用手順）

リスクと対策（個人攻撃回避のポリシー）

効果（調査時間短縮、再発防止への接続）

### タイトル案

「“犯人探し”をやめて証跡を残す：blame運用設計」

「監査に耐える変更履歴：blame/ignore-rev/CODEOWNERSの三点セット」

「最終更新者は誰か、を1秒で出すCLIを作った話」

### 捨てるもの
コマンド羅列。代わりに設計意図と出力例1枚。

### 0. はじめに

> このファイルのこの行だけ他の同様のファイルと記述が統一されていませんが、これはどういう意図でしょうか？

**知らんがな。**
保守対応を続けているとご新規さんからこういう感じの指摘がよく入るのですが、**私は開発者ではないので知りません。**

しかし攻撃的になって対応するとチーム内で不協和音になってしまいます。

なるべくオブラートにしつつ説明責任は回避したいところですね。

そこで役立つのが`git blame`と`git log`です。

あ、SVNには使えないんで悪しからず。

### 1. 簡単git blame

```bash
git blame -L 48,52 path/to/file
```

```bash
011aa751 (Author 2025-08-20 23:44:50 +0900 48) try:
011aa751 (Author 2025-08-20 23:44:50 +0900 49)     from aws_secretsmanager_caching import SecretCache, SecretCacheConfig
011aa751 (Author 2025-08-20 23:44:50 +0900 50) except Exception:  # pragma: no cover - lib optional
011aa751 (Author 2025-08-20 23:44:50 +0900 51)     SecretCache = None  # type: ignore
011aa751 (Author 2025-08-20 23:44:50 +0900 52)     SecretCacheConfig = None  # type: ignore
```

簡単に履歴・作成者・ソースコードの行が出てきて、見た目のインパクトが抜群です。
しかし、ここで出てくるのは**作成者**であって、更新者ではないのですね。

### 2. コピペ用git blame

**更新者**を出したい場合は`--line-porcelain`オプションを使用します。

```bash
git blame --line-porcelain -L 120,150 path/to/file  |
awk '
  /^[0-9a-f]{40} /   {sha=$1}
  /^committer /      {name=substr($0,11)}
  /^committer-mail / {mail=substr($0,16)}
  /^committer-time / {t=$2}
  /^committer-tz /   {tz=$2}
  /^\t/ {
    cmd = "date -d @" t " +%Y-%m-%dT%H:%M:%S%:z"
    cmd | getline iso; close(cmd)
    printf "%s  %s  %s  %s  %s\n", iso, sha, name, mail, substr($0,2)
  }'
```

```bash:コマンド結果の一例
2025-08-22T23:44:50+09:00  5d33ab91eb011aa7515b474b2669dac17fa9b2b9  Commmiter  <commiter@sample.com>  - Config models (dataclasses)
```

#### 解説

`git blame`はデフォルトではファイル作成者の情報しか出さない。
変更者の情報を出す場合は`--line-porcelain`を

### 出典

@[card](https://git-scm.com/docs/git-blame)