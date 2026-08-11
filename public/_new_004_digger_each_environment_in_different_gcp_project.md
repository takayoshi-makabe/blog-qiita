---
title: Diggerを使って各環境ごとに異なるGCPプロジェクトでTerraformを実行する
tags:
  - Terraform
  - GitHubActions
  - Digger
private: false
updated_at: '2025-06-06T12:20:58+09:00'
id: d0206cc5c356023c0561
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# Diggerについて

日本語での情報はあまりないですが、こちらの方の記事が比較的まとまっていて参考になると思います！

- [Digger + GitHub Actionsで作るTerraform/OpenTofuのCI/CD](https://qiita.com/minamijoyo/items/b61806b570d9d1257f0b)

サーバレスでAtlantis風のApply-Before-MergeスタイルのCI/CDが手軽に実現できるのがDiggerの特徴です。

# 導入して最初に困ったところ

ありふれたディレクトリ構成ですが、以下の様に定義していました。尚、本番・開発でGCPプロジェクトを分けている前提です。

``` shell
.
├── .github
│   └── workflows
│       └── digger_workflow.yml
├── .terraform-version
├── digger.yml
└── terraform
    ├── environments
    │   ├── dev
    │   └── prod
    └── modules
```

`terraform/environments`の下に環境ごとのディレクトリを作成し、共通モジュールを`terraform/modules`に配置しています。そして、Diggerの設定ファイル`digger.yml`のプロジェクト定義を以下のようにしていました。

``` yaml
# digger.yml
projects:
  - name: dev
    dir: terraform/environments/dev
    workflow: dev
    include_patterns:
      - terraform/environments/dev/**
      - terraform/modules/**
  - name: prod
    dir: terraform/environments/prod
    workflow: prod
    include_patterns:
      - terraform/environments/prod/**
      - terraform/modules/**
```

また、ワークフローにおける Digger 実行ステップは以下のように定義していました。

``` yaml
# .github/workflows/digger_workflow.yml
name: Digger (Terraform on GitHub Actions)

on:
  pull_request:
    paths:
      - terraform/**
    types: [opened, synchronize]
  issue_comment:
    types: [created]

concurrency:
  group: digger-run

permissions:
  contents: write      # required to merge PRs
  actions: write       # required for plan persistence
  id-token: write      # required for workload-identity-federation
  pull-requests: write # required to post PR comments
  issues: read         # required to check if PR number is an issue or not
  statuses: write      # required to validate combined PR status

jobs:
  digger-dev:
    if: |
      github.event_name == 'pull_request' ||
      (
        github.event_name == 'issue_comment' &&
        github.event.issue.pull_request != null && (
          startsWith(github.event.comment.body, 'digger apply') ||
          startsWith(github.event.comment.body, 'digger plan')
        )
      )

    environment: dev
    name: Digger(Development)
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Read TF version
        id: tfversion
        run: |
          echo "TF_VERSION=$(cat .terraform-version)" >> "$GITHUB_OUTPUT"

      - name: Google Auth (Workflow identity Federation)
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}
          service_account: ${{ secrets.GCP_DIGGER_SA }}
          token_format: access_token
          create_credentials_file: true
          export_environment_variables: true

      - name: Run Digger
        uses: diggerhq/digger@v0.6.100
        with:
          no-backend: true
          disable-locking: true
          cache-dependencies: true
          terraform-version: ${{ steps.tfversion.outputs.TF_VERSION }}
          google-lock-bucket: 'gcp'
          upload-plan-destination-gcp-bucket: ${{ secrets.GCP_DIGGER_BUCKET }}
          setup-google-cloud: false
          setup-terraform: true
        env:
          GITHUB_CONTEXT: ${{ toJson(github) }}
          GITHUB_TOKEN:   ${{ secrets.GITHUB_TOKEN }}

  digger-prod:
    if: |
      github.event_name == 'pull_request' ||
      (
        github.event_name == 'issue_comment' &&
        github.event.issue.pull_request != null && (
          startsWith(github.event.comment.body, 'digger apply') ||
          startsWith(github.event.comment.body, 'digger plan')
        )
      )

    environment: prod
    name: Digger(Production)
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Read TF version
        id: tfversion
        run: |
          echo "TF_VERSION=$(cat .terraform-version)" >> "$GITHUB_OUTPUT"

      - name: Google Auth (Workflow identity Federation)
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}
          service_account: ${{ secrets.GCP_DIGGER_SA }}
          token_format: access_token
          create_credentials_file: true
          export_environment_variables: true

      - name: Run Digger
        uses: diggerhq/digger@v0.6.100
        with:
          no-backend: true
          disable-locking: true
          cache-dependencies: true
          terraform-version: ${{ steps.tfversion.outputs.TF_VERSION }}
          google-lock-bucket: 'gcp'
          upload-plan-destination-gcp-bucket: ${{ secrets.GCP_DIGGER_BUCKET }}
          setup-google-cloud: false
          setup-terraform: true
        env:
          GITHUB_CONTEXT: ${{ toJson(github) }}
          GITHUB_TOKEN:   ${{ secrets.GITHUB_TOKEN }}
```

- Workload Identity Federation を使ってGCPのサービスアカウントに認証。サービスアカウントキーをGitHub Secretsに保存する必要がなく、セキュア。
- GitHubのEnvironment Secretsを環境ごとに異なるSecretsを設定できるので、GCPのプロジェクトごとに異なる認証情報を設定可能。
- `.terraform-version` ファイルを使って、プロジェクトでTerraformバージョンを統一。

で、このままやっていこうとすると以下の様なエラーが出てしまいました。


※ digger-dev のエラー

![004-digger-prod-error.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/ebd35d53-6edb-40e6-b2d8-b191d2bf8312.png)

※ digger-prod のエラー

![004-digger-dev-error.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/ca896285-5615-44e8-8b1a-ccae470ac047.png)

何が起こっているかというと、

- digger-dev → 開発環境に Terraform **成功**
- digger-dev → 本番環境に Terraform **権限不足で失敗**
- digger-prod → 開発環境に Terraform **権限不足で失敗**
- digger-prod → 本番環境に Terraform **成功**

のようになっていました。

# 解決方法

日本語ドキュメントも少なく苦戦しましたが、公式を読み漁っていると解決方法にたどり着きました。

参考：[公式ドキュメント](https://docs.digger.dev/ce/reference/digger.yml)

```
引用:
Note: you can also name your Digger configuration file differently, and specify its name using the digger-filename input at GitHub Action level.
```

つまり、Diggerの設定ファイルを環境ごとに分けて、GitHub Actionsのワークフローでそれぞれの環境に対応する設定ファイルを指定することで解決できます。

``` yaml
# digger-dev.yml
projects:
  - name: dev
    dir: terraform/environments/dev
    workflow: dev
    include_patterns:
      - terraform/environments/dev/**
      - terraform/modules/**
```

``` yaml
# digger-prod.yml
projects:
  - name: prod
    dir: terraform/environments/prod
    workflow: prod
    include_patterns:
      - terraform/environments/prod/**
      - terraform/modules/**
```
そして、GitHub Actionsのワークフローでそれぞれの環境に対応する設定ファイルを指定します。

``` yaml
# .github/workflows/digger_workflow.yml
      - name: Run Digger (Development)
        with:
          digger-filename: digger-dev.yml  # 開発環境用の設定ファイルを指定

      - name: Run Digger (Production)
        with:
          digger-filename: digger-prod.yml  # 本番環境用の設定ファイルを指定
```

これで、各環境ごとに異なるGCPプロジェクトを使用してTerraformを実行できるようになりました。
