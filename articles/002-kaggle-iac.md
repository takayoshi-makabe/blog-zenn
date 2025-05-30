---
title: "Terraformを使ってGoogleCloud上にKaggle環境をささっと構築する"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "Terraform"
  - "Kaggle"
  - "Docker"
  - "Digger"
published: true
---

# はじめに

サクッと Kaggle 環境を構築したいときってありますよね。今回は Terraform を使って Google Cloud Platform 上に Kaggle 環境を構築する方法を紹介します。この記事を読むことで、以下のような複数のインスタンスを立ち上げることができるようになります。

- 初めから Kaggle API が使える状態
- 初めから Docker が使える状態
- 初めから GitHub が使える状態
- GCS へのアクセス権限がある状態
- 静的 IP アドレスが割り当てられている状態

以下、リポジトリになります。

[GitHub Repos](https://github.com/spider-man-tm/kaggle-infrastructure)

# GCP Project のセットアップ

まず始めに、GCP のプロジェクトを作成します。プロジェクト名は任意ですが、この記事では `kaggle-infra` としています。必要なコマンドは [setup-gcp-project/Makefile](https://github.com/spider-man-tm/kaggle-infrastructure/blob/main/setup-gcp-project/Makefile) にまとめているので以下のコマンドを叩けば最低限必要な準備ができた状態でプロジェクトが作成されます。

```bash
cd setup-gcp-project
make setup-gcp-project \
    GCP_PROJECT_ID=<your_gcp_project_id> \
    BILLING_ACCOUNT_ID=<your_billing_account_id> \
    KAGGLE_KEY=<your_kaggle_api_key> \
    TF_STATE_BUCKET_NAME=<your_tf_state_bucket_name>
```

一応、やっていることをざっと説明すると

- GCP プロジェクトの作成と billing account の紐付け
- 必要な API の有効化
- tfstate 用の GCS バケットの作成
- シークレットマネージャーの作成

最後のシークレットマネージャーですが、この後の手順で作成するインスタンスに対して、Kaggle API キーを渡すために使います。terraform.tfvars に直接書くのはセキュリティ的によくないので、シークレットマネージャーに格納しておきます。

# Terraform でインフラを構築

Terraform では必要最低限のリソースをカバーするためのモジュールをまとめた`modules/`ディレクトリと、複数コンペを同時並行で走らせたり、環境を各々切り分けるために`environments/`ディレクトリを用意しています。
動かし方は README にも記載していますが、まず始めに自身が用意した環境、例えば`terraform/environments/competition01/`配下に`terraform.tfvars`ファイルを作成し、そちらに必要な変数群を定義します。

```hcl
project_id = "kaggle-infra-exp-0808"
region     = "asia-northeast1"

# GCE
zone             = "asia-northeast1-c"
instance_name    = "instance-kaggle-demo"
machine_type     = "e2-micro"
image            = "ubuntu-os-cloud/ubuntu-2004-lts"
network_name     = "default" # VPC Network name
github_email     = "abc.defg@gmail.com"
github_user_name = "spider-man-tm"
kaggle_username  = "kaggleusername" # Your Kaggle username

# GCE & Network
instance_count = 2

# Network
static_ip_name = "static-ip-kaggle-demo"

# GCS
tf_state_bucket_name    = "tf-state-kaggle-infra-exp-0808"
digger_bucket_name      = "digger-bucket-kaggle-infra-exp-0808"
competition_bucket_name = "competition-bucket-kaggle-infra-exp-0808"
location                = "ASIA"

# Github Actions
github_repo_name = "kaggle-infrastructure"
```

次に、`terraform/environments/competition01/terraform.tf` ファイルに上のコマンドで作成した、tfstate 用の GCS バケットを指定します。prefix は適宜変更してください。

```hcl
terraform {
  required_version = ">= 1.6"

  backend "gcs" {
    bucket = "tf-state-bucket-name-titanic"  # <- GCS Bucket name
    prefix = "terraform/state"
  }

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.30"
    }
  }
}
```

作成するインスタンスですは、GitHub のユーザー情報と、Kaggle User Name の情報を定義しています。[terraform/modules/gce/main.tf](https://github.com/spider-man-tm/kaggle-infrastructure/blob/main/terraform/modules/gce/main.tf) と
[terraform/modules/gce/startup-script.sh.tpl](https://github.com/spider-man-tm/kaggle-infrastructure/blob/main/terraform/modules/gce/startup-script.sh.tpl) を見るとわかりますが、docker コマンドのインストール、GitHub のセットアップ、Kaggle API のセットアップ等を行なった状態で GCE が起動するようにしています。ただし、Kaggle API に利用する Kaagle Key だけは一応プライベートなものなので、先ほど GCP プロジェクトの立ち上げ時に作成した Secret Manager に対して、[data block](https://github.com/spider-man-tm/kaggle-infrastructure/blob/main/terraform/environments/competition01/main.tf) を使って参照する形にしています。

```hcl
# Get the value from secret manager
data "google_secret_manager_secret_version" "kaggle_key" {
  secret  = "kaggle-key"
  version = "latest"
}
```

# Docker

一応、インスタンスを起動した後に簡単にコンテナも立ち上げられるようにしたかったので、`docker/`ディレクトリ以下に必要なファイルをまとめています。

```shell
cd docker
docker-compose up -d
```

# Digger

ローカル環境からリソースの作成を行うことも可能ですが、Digger を使うことで GitHub Actions 上で Terraform を実行することができます。
準備方法は README にも記載していますが、リポジトリに Environment Secrets を設定する必要があります。

※ コンペティション毎に用意する環境を用意

![](/images/002/img01.png)

※ 用意された環境に紐づく Secrets を設定

![](/images/002/img02.png)

こうすることで、PR 作成時に自動的に `terraform plan` が実行され、PR のコメントに結果が表示されます。また PR のコメントに `digger plan`・`digger apply` とコメントすることで、GitHub Actions が `terraform plan`・`terraform apply` を実行します。

参考：[https://github.com/spider-man-tm/kaggle-infrastructure/pull/10](https://github.com/spider-man-tm/kaggle-infrastructure/pull/10)

# 終わりに

まだまだ作りかけなので徐々に追加予定ですが、そもそもここ 3 年くらい Kaggle に取り組めてないのでそろそろ再開したい今日この頃です。
