---
title: "最新Gitコマンドの洗礼を受けて知識アップデート"
emoji: "🌱"
type: "tech"
topics: ["bash", "linux", "cli"]
published: false
---

## Gitコマンド備忘録

### 0. はじめに
新規ブランチ作成時に`git checkout -b`を使っていたら2005年のコマンドって揶揄された

しょうがないので

#### 新規プランチ作成・追跡・プッシュ

```bash
# 新規ブランチ作成
git switch -c new_branch

# 追跡・プッシュ
git push -u new_branch
```
`-u:`「このローカルブランチは `origin/new_branch` を追いかけます」という意味
これ以降いちいち`git push origin dev`とか打たなくても
`git pull`や`git push`だけで通じる
