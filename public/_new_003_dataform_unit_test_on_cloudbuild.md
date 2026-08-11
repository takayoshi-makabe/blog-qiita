---
title: Dataformの単体テストをCloud Buildで実行する
tags:
  - Terraform
  - CloudBuild
  - CICD
  - Dataform
private: false
updated_at: '2025-06-06T14:53:39+09:00'
id: 360fa13724807f509820
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# GitHub Actions を使う場合

Dataform CLI では単体テストを実施することができます。普段はassertion機能がメインで単体テストは使ってないというユーザーも多いかと思います。確かに最低限のデータ品質という意味では assertion で十分かもしれませんが、クエリが複雑化してきたり、運用フェーズでクエリがバージョンアップを繰り返してくると、ちゃんとテストを書いておくことで、ロジックが壊れていないかを確認することができます。

そこで運用の中でテストの自動実行をしていこうと考えた際に、GitHub Actions を使ったテスト自動化の方法を調べていくと以下のようなブログにたどり着きました。

[Github ActionsでDataformをデプロイする方法](https://qiita.com/leemunhui/items/d2a9f060341113b00c07)

記事の内容ですが、GitHub Actions から Dataform をデプロイする方法として以下のように記載されています。

- Dataform CLI を叩く方法と、API を使う方法の2つがある
- 前者は credential 情報を暗号化しているとはいえリポジトリ上で管理するというデメリットがある
- 一方、単体テストは API が用意されておらず、CLIを叩くしかない

となると、credentialをリポ管理するというリスクやサービスアカウントキーの管理などの問題が発生してしまいます。特に人数が多くなると鍵の管理は結構面倒な問題です。

※ 補足
ローカル環境で`dataform init-creds`を叩いて、`.df-credentials.json`を作成する場合、ADC(Application Default Credential)を使う方法と、サービスアカウントキーを使う方法があります。上記の記事で使用している`.df-credentials.json`はGitHub Actionsで使うものになるので、サービスアカウントキーを使う方法になります。（なので暗号化は必須）

# Cloud Build を使う場合

いきなり結論ですが、Cloud Buildを使うことで、上の問題を解決してしまおうという試みが今回のブログのテーマです。

まず、Dataformと同じプロジェクト内に必要なロールを付与したサービスアカウントを作成します。

```hcl
# サービスアカウントの作成
resource "google_service_account" "dataform_unit_test" {
  project      = var.project_name
  account_id   = "dataform-unit-test"
  display_name = "dataform-unit-test"
  description  = "Dataform単体テスト用のサービスアカウント"
}

# サービスアカウントに付与するロールを作成
resource "google_project_iam_member" "bigquery_job_user_member" {
  project = var.project_name
  role    = "roles/bigquery.jobUser"
  member  = "serviceAccount:${google_service_account.dataform_unit_test.email}"
}
resource "google_project_iam_member" "logs_writer_member" {
  project = var.project_name
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.dataform_unit_test.email}"
}
```

次に、このサービスアカウントを使って Cloud Build Trigger を作成します。

```hcl
resource "google_cloudbuild_trigger" "dataform_unit_test" {
  name        = "dataform-unit-test"
  description = "Dataformの単体テストを実行する"

  github {
    owner = "hoge"
    name  = "repo-name"
    pull_request {
      # ^main$
      # ^develop$
      # ^(main|develop)$ などのように複数のブランチを指定することも可能
      branch = "^saga$"
    }
  }
  filename           = "cloudbuild/dataform-test.yml"
  include_build_logs = "INCLUDE_BUILD_LOGS_WITH_STATUS"
  service_account    = google_service_account.dataform_unit_test.id
}
```

こうすることで、指定したリポジトリの指定したブランチに対して、PRが発行されたタイミングで`repo-name/cloudbuild/dataform-test.yml`で定義した単体テストを実行するように設定できます。

後は、レポジトリ直下に`cloudbuild`ディレクトリを作成し、その中に`dataform-test.yml`を作成します。

```yaml
steps:
  - id: Run dataform unit test
    name: node:lts
    entrypoint: bash
    args:
      - "-c"
      - |
        echo '{"projectId": "'${PROJECT_ID}'", "location": "US"}' > .df-credentials.json
        npm i -g --quiet @dataform/cli@3.0.0
        dataform install
        dataform test

timeout: 60s
options:
  logging: CLOUD_LOGGING_ONLY
```

※ 補足
先ほど紹介した GitHub Actions で使用する`.df-credentials.json`は、サービスアカウントキーを使用する関係上、認証情報を含んでいましたが、今回は必要ロールを付与した Cloud Build で実行するため、認証情報を含んでいません。よって以下のjsonをリポジトリ直下に出力することで事足りています。

```json
{
  "projectId": "your-project-id",
  "location": "US"
}
```
