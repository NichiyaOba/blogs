## はじめに

ご無沙汰しております。LRMでエンジニアをしている大場です。

本記事ではTerraformを使用してNew Relicアラートを定義し、異常検知の設定を継続的に更新・運用していく仕組み作りを紹介します。

## 背景

先日、社内の勉強会にてNew Relicを扱った際、監視項目の設定やアラート設定管理の属人化が課題に上がりました。

LRMでは最近、インフラ構成管理のTerraform化とTerraform Cloudによる管理を進めています。そんな中、勉強会中の調査でTerraformを利用したNew Relicアラート定義が可能というドキュメントを見つけました。

アラートをTerraformでコード化しアプリケーション側のリポジトリで管理することで、リポジトリ内のアプリケーションのコードとアラート定義のコード両方をAIに参照させることができます。

AIとのバイブコーディングによりキャッチアップのハードルを下げること、また既存のアプリケーションの特徴を基に最適なアラート定義を導き出せる状態を整えることが狙いです。

## 概要

そもそもTerraformって何？という方は下記参考にしていただければと思います。AWS S3をTerraformで作成する簡単なハンズオンと合わせて紹介しています。

https://zenn.dev/lrm/articles/578380b7a9091d

New Relicは、アプリケーションのパフォーマンスをリアルタイムで監視・分析でき、アラートの定義が可能です。アラートでは事前に決めた条件にマッチした際の通知が可能で、条件は詳細に設定することができます。

https://docs.newrelic.com/jp/docs/alerts/overview/

この設定はNew RelicのWeb UI上からも可能ですが、今回はTerraform Cloudを介して反映する手順を紹介します。
弊社の多くのアプリケーションはモノレポな構成になっており、frontendとbackendのコードを1つのリポジトリにまとめて管理しています。

このリポジトリの中に、アラート定義用のフォルダとして`ops/newrelic`フォルダを作成し、その中に.tfファイルを配置します。

```bash
project/
├── frontend/
├── backend/          
├── ops/
│   └── newrelic/
│       └── backend/
│           ├── alerts.tf 🆕
│           ├── providers.tf 🆕
│           ├── variables.tf 🆕
│           └── versions.tf 🆕
・・・　その他フォルダ・・・
```

今回導入したチームでは、Next.jsのフロントエンドとGoのバックエンドを1つのリポジトリ（モノレポ）で管理しています。

以下、実際の作業をまとめます。

## 作業手順

以下二つのセクションに分けて記述します。

1. .tfファイルで必要なリソースを定義する
2. Terraform Cloud上でProject / Workspaceを作成する

### 1.tfファイルで必要なリソースを定義する

https://docs.newrelic.com/docs/infrastructure-as-code/terraform/terraform-intro

基本は上記手順に則っていますが、1点違うのはmain.tfを作成していない点です。moduleのインポートや複雑なフォルダを跨いだ参照はしておらず、簡素な構成なので不要と判断しました。
backend向けの監視内容は以下のフォルダ構成で実装しました。

```bash
project/
├── frontend/
├── backend/          
├── ops/
│   └── newrelic/
│       └── backend/
│           ├── alerts.tf 🆕
│           ├── providers.tf 🆕
│           ├── variables.tf 🆕
│           └── versions.tf 🆕
・・・　その他フォルダ・・・
```

それぞれのファイルは以下の通り定義しました。

``` terraform:versions.tf

terraform {
  required_version = "~> 1.0"

  required_providers {
    newrelic = {
      source  = "newrelic/newrelic"
    }
  }
}

```

``` terraform:variables.tf
variable "NEWRELIC_API_KEY" {
  type        = string
  description = "New Relic User API key"
  sensitive   = true
  validation {
    condition     = length(var.NEWRELIC_API_KEY) > 0
    error_message = "New Relic API key must not be empty"
  }
}

variable "NEWRELIC_ACCOUNT_ID" {
  type        = number
  description = "New Relic account id"
  validation {
    condition     = var.NEWRELIC_ACCOUNT_ID > 0
    error_message = "New Relic account id must be a positive number"
  }
}

variable "NEWRELIC_REGION" {
  type        = string
  description = "New Relic region, either US or EU"
  default     = "US"
}

variable "NEWRELIC_ALERT_POLICY_NAME" {
  type        = string
  description = "New Relic alert policy name"
  default     = "application"
}

```

```terraform:providers.tf
provider "newrelic" {
  account_id = var.NEWRELIC_ACCOUNT_ID
  api_key    = var.NEWRELIC_API_KEY
  region     = var.NEWRELIC_REGION
}
```

``` terraform:alerts.tf
resource "newrelic_alert_policy" "{エンティティ名}" {
  name                = var.NEWRELIC_ALERT_POLICY_NAME
  incident_preference = "PER_CONDITION"
}

resource "newrelic_nrql_alert_condition" "application_error_rate" {
  account_id = var.newrelic_account_id
  policy_id = newrelic_alert_policy.application.id
  type = "static"
  name = "${var.newrelic_alert_policy_name} - Error rate"
  enabled = true
  violation_time_limit_seconds = 259200

  nrql {
    query = "SELECT (count(apm.service.error.count) / count(apm.service.transaction.duration)) * 100 AS 'Error %' FROM Metric WHERE appName = '{エンティティ名}'"
    data_account_id = var.newrelic_account_id
  }
  critical {
    operator = "above"
    threshold = 22
    threshold_duration = 300
    threshold_occurrences = "all"
  }
  fill_option = "none"
  aggregation_window = 60
  aggregation_method = "event_flow"
  aggregation_delay = 120
}
```

アラートはあくまでサンプルになります。内容としては1分間隔で集計して5分間継続でエラーレートが22%を超えたらアラートを発報するというものです。

`appName = {エンティティ名}`の部分を環境変数で入れるか悩みましたが、運用としてNew RelicのNRQLをそのままコピペする方が環境変数に書き換える手間が減るので良いと判断しました。アプリケーション側のエンジニアに依頼するのは以下の4ステップで済みます。

1. 監視対象にしたいNRQLをNew Relic上で調べて`alerts.tf`の`nrql`プロパティにコピペ
2. 適切なアラート条件を設定する
3. PR立ててレビュー
4. developにマージ

NRQLの内容に関しては以下のリンク先にある通り、グラフ形式で表示されているタブからもNRQLを直接取得できます。

https://docs.newrelic.com/jp/docs/nrql/get-started/introduction-nrql-new-relics-query-language/

### 2.Terraform Cloud上でProject / Workspaceを作成する

続いて、Terraform Cloud上での設定について記述します。前提としてTerraform CloudとGithubを連携しておく必要があります。具体的な手順は[こちら](https://developer.hashicorp.com/terraform/cloud-docs/vcs/github-app)をご確認ください。

Gihubの連携が終わったらまずは右上のNewのボタンからProjectを作成します。

![](https://storage.googleapis.com/zenn-user-upload/204eb54d3433-20251001.png)

続いてGit管理するために一番左のVersion Control Workflowを選択

![](https://storage.googleapis.com/zenn-user-upload/42c2a596a530-20251001.png)

GitHubAppを選択

![](https://storage.googleapis.com/zenn-user-upload/eb3ff6c1f256-20251001.png)

Select VCS Repositoryから対象のリポジトリを選択

![](https://storage.googleapis.com/zenn-user-upload/e21d48ed3eb3-20251001.png)

Workspace Nameを入力したらAdvanced optionsを設定していきます

![](https://storage.googleapis.com/zenn-user-upload/15efdde6ff94-20251001.png)

今回、実行対象の.tfファイルは/newrelic/backend配下にあるのでTerraform Working Directoryにはそのパスを入力します。
Auto-applyはどちらも許容して、Plan実行時に問題がなければそのまま自動Applyします。
VCS-Branchはdefaultに設定されたブランチに対して変更が入ったらApplyしたいので空欄にしました。

![](https://storage.googleapis.com/zenn-user-upload/625791f8ff1b-20251016.png)

Automatic Run triggeringでは、`ops/newrelic`配下のファイルに変更があった時だけ検知するようにします。
また、Automatic speculative plansをONにしておくとPR作成時に差分をTerraform のPlanで確認できます

![](https://storage.googleapis.com/zenn-user-upload/f18cb86fefe6-20251008.png)

最後はCreateを押して設定完了です。

実際にdefault指定したブランチに対してPRを立てると以下の通りPlanが実行されてリソース変更がコンソール上から確認できます。defaultブランチにマージ後、今回の設定では自動Applyされます。
![](https://storage.googleapis.com/zenn-user-upload/adbec085bcac-20251002.png)

## 結果

デフォルトブランチにNew Relicのアラート定義変更が取り込まれるたびにアラートが自動適用されるようになりました。また、単にアラート定義が自動化されるだけでなく、コードベースでAIのレビューを通すことが出来るようになったことは非常に大きなメリットです。

既存のソースコードをコンテキストとして取り込んだ上でAIが回答してくれるので、今までNew Relicアラートを組んだことがない人でもAIとの対話で適切なアラート定義ができそうです。

実際に、既存のアラートをそのまま移植した部分でCodeRabbitから的確な指摘を貰いました。

![](https://storage.googleapis.com/zenn-user-upload/79f31585f78f-20251003.png)

## 終わりに

アプリケーションコードの変化と同時にアラートも都度メンテナンスして最適化できるので運用上非常に良い構成になったと思っています。今後運用に乗せていけるよう取り組みます。

これからも小さな改善を重ねていきたいと思います。ここまで目を通して頂きありがとうございました。