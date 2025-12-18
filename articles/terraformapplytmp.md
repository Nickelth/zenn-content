---
title: "CloudShellの$HOMEでTerraformすると容量が足りないので /tmp 配下で展開して回す"
emoji: "🧯"
type: "tech"
topics: ["aws","cloudshell","terraform","iac"]
published: true
---

AWS CloudShell は便利です。ブラウザでサクッと AWS CLI / Terraform を叩けるので、検証や“薄切り”の IaC と好相性。

ただし、**CloudShell の $HOME は 1GB**です。Terraform は init しただけで `.terraform/` が太りやすく、気づくと **`No space left on device`** で詰みます。 

以前ポートフォリオ作成用にTerraformを展開しようとしましたが、これが原因で詰んでしまいました。

この記事では、**CloudShell で Terraform を回すときは /tmp 配下に作業ディレクトリを展開して実行する**、という運用に寄せて事故を避ける方法をまとめます。

## 何が起きるのか（なぜ $HOME が死ぬのか）

CloudShell の永続ストレージは **$HOME 配下 1GB**で、増やせません。   
Terraform は `terraform init` で provider を取得し、作業ディレクトリ配下に `.terraform/` を作ります。

- provider バイナリはそこそこ大きい
- 構成が分割されている（ディレクトリが複数）と、`init` のたびに各ディレクトリに `.terraform/` ができる
- module が多いほど「気づいたら増えてる」

実際、Terraform 側でも「provider が各ディレクトリに展開されてディスクを食う」系の話は昔から出ています。 

## 現状確認

CloudShell で以下を叩いて、**どこが太っているか**確認します。

```bash
df -h
du -h -d 2 "$HOME" | sort -h | tail -n 20
du -hs .terraform 2>/dev/null || true
```

だいたい `.terraform/` か、証跡ログを雑に溜めたディレクトリが犯人です。

## 方針：Terraform の作業ディレクトリを /tmp に逃がす

狙いはシンプルです。

* **作業ツリー（= .terraform が増える場所）を /tmp に置く**
* 逆に、残したいもの（証跡ログなど）は $HOME 側へ戻す（もしくは S3）

CloudShell の制限は “$HOME の永続 1GB” が本体なので、ここを温存します。 ([AWS ドキュメント][1])

## 手順（手動版）

### 1) リポジトリを /tmp に展開

```bash
# 例: $HOME に repo がある前提
REPO="$HOME/portfolio"
WORK="/tmp/tf-$(date +%Y%m%d-%H%M%S)"

mkdir -p "$WORK"
rsync -a --delete "$REPO/" "$WORK/repo/"
cd "$WORK/repo"
```

### 2) Terraform は /tmp 側で実行

例として `infra/00-network` と `infra/30-ecs-alb-mlops` の “薄切り”適用を想定します。

```bash
cd infra/00-network
terraform init
terraform plan -target=module.network
terraform apply -target=module.network

cd ../30-ecs-alb-mlops
terraform init
terraform plan -target=module.ecs
terraform apply -target=module.ecs
```

### 3) 証跡を $HOME に回収（任意だけど推奨）

```bash
# 証跡置き場（例）
EVID="$HOME/portfolio/docs/evidence"
mkdir -p "$EVID"

# /tmp のログを回収（例）
find "$WORK/repo/docs/evidence" -type f -maxdepth 1 -name '*_tf_*.txt' -print -exec cp -a {} "$EVID/" \;
```

### 4) /tmp を掃除

```bash
rm -rf "$WORK"
```

## もう少し快適にする：plugin cache を使う（任意）

Terraform には provider のダウンロードを節約する **plugin cache** の仕組みがあります。設定は `plugin_cache_dir`（CLI設定）で指定できます。 ([HashiCorp Developer][2])

CloudShell では、**plugin cache も /tmp に置く**のが安全です（$HOME を太らせない）。

```bash
export TF_CLI_CONFIG_FILE="$HOME/.terraformrc"
cat > "$TF_CLI_CONFIG_FILE" <<'EOF'
plugin_cache_dir = "/tmp/terraform-plugin-cache"
EOF

mkdir -p /tmp/terraform-plugin-cache
```

> 注意: plugin cache は「provider の取得」を助けますが、作業ディレクトリ側の `.terraform/` 生成自体が消えるわけではありません。なので結局、**作業ディレクトリを /tmp に置く方が効きます**。

## 実例：CloudShell で薄切り IaC を回したときの構成メモ

自分のケースでは、以下の整理をして `-target` で段階適用できる状態にしました。

* ディレクトリ分割：`infra/00-network`, `20-ecr`, `30-ecs-alb-mlops`
* `00-network`

  * ALB / Listener(80) / TargetGroup(`/health`) / Logs / SG(Alb↔Tasks:8000)
  * outputs: `alb_dns`, `tg_arn`, `tasks_security_group_id`, `log_group_name`
* `30-ecs-alb-mlops`

  * ECS Cluster / Service(回路遮断 + HCG=60s) / TaskDefinition
  * TaskRole を作り、S3 `GetObject`（モデル取得）を付与
* `terraform plan -target=module.network → apply`
* `terraform plan -target=module.ecs → apply`

apply 後に `alb_dns` を outputs から引いて、ALB/TG が Healthy であることを確認（`/health`）まで到達できました。

## よくある落とし穴

* **/tmp は永続ではない**
  セッション次第で消える前提にして、残したい証跡は $HOME か S3 に逃がします。CloudShell の永続ストレージは 1GB で固定です。 ([AWS ドキュメント][1])
* **“$HOME で init した残骸”が残っている**
  `.terraform/` を消すだけで復活することも多いです。
* **何が太っているか見ずに悩む**
  `df -h` と `du` を叩くのが最短です。

## まとめ

* CloudShell の $HOME は **1GB 固定**で、Terraform を素直に回すと詰みやすい。 ([AWS ドキュメント][1])
* 対策はシンプルで、**/tmp 配下に作業ディレクトリを展開して Terraform を実行する**
* 証跡や残したいものだけ $HOME（または S3）に回収する
* 余裕があれば `plugin_cache_dir` で provider の取得を軽くする ([HashiCorp Developer][2])

[1]: https://docs.aws.amazon.com/ja_jp/cloudshell/latest/userguide/limits.html "のサービスクォータと制限 AWS CloudShell"
[2]: https://developer.hashicorp.com/terraform/cli/config/config-file "Create a Terraform CLI configuration file"
