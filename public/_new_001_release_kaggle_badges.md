---
title: GitHubプロフィールにKaggleのバッジを表示するGitHub Actionsを作成しました
tags:
  - TypeScript
  - Kaggle
  - GitHubActions
private: false
updated_at: "2025-06-06T14:53:39+09:00"
id: 8e287bc6a8f90018049c
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

こんにちは、[@Takayoshi_ma](https://x.com/Takayoshi_ma)です。最近は専ら遠ざかっていますが、以前までは精力的に Kaggle に取り組んでいました。

本日ですが、GitHub のプロフィールに Kaggle のバッジを表示する GitHub Actions を作成しましたので、その紹介をさせていただきます。もしこのプロジェクトにご興味あれば是非とも GitHub でスターをお願いします ⭐️

- [GitHub Repository](https://github.com/takayoshi-makabe/kaggle-badges)

# 作成した GitHub Actions

- [GitHub Actions Marketplace](https://github.com/marketplace/actions/kaggle-badges)

README にも記載していますが、この GitHub Actions は Kaggle のユーザー名を指定すると、Kaggle ランクに応じたバッジを自動スケジューリングで生成してくれるものです。生成されたバッジはレポジトリ直下に push されるので、その URL を使って README などに貼り付けることで、Kaggle の称号を GitHub のプロフィールに表示することができます。

(生成されるバッジの例)

以下はコンペティション用のバッジのリストです。同様のスタイルのバッジがデータセット、ノートブック、ディスカッションでも自動生成されます。添付画像だと分かりづらいですが、下段のプレートはアニメーション付きです。

![.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/8523f4b4-6aa6-7a0c-cb23-635082589346.png)

(GitHub のプロフィールに表示される例)

![my-github-profile.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/68cfa3b2-fbec-da83-144b-af1fd4ad819b.png)

# 使い方

[README](https://github.com/takayoshi-makabe/kaggle-badges)にも記載していますが、使い方は以下の通りです。

## 1. 専用リポジトリを作成する

GitHub では自身の GitHub アカウント名と同じリポジトリを作成すると、そのリポジトリが GitHub のプロフィールに表示される機能があります。

- 参考：[Managing your profile README](https://docs.github.com/ja/account-and-profile/setting-up-and-managing-your-github-profile/customizing-your-profile/managing-your-profile-readme)

このリポジトリ直下に`README.md`を作成すると、その内容がプロフィールに表示されます。今後のステップでこのリポジトリに Kaggle のバッジを表示するためのバッジを生成する GitHub Actions を設定します。

## 2. ワークフローの権限設定

上記で作成したリポジトリにワークフローを設定していきますが、その前に権限設定を行います。
`Settings > Actions > General > Workflow Permissions > Read and write access` から設定を変更してください。

![workflow-permissions.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/332b90de-48ec-b95d-4acd-4b7ad7c1d862.png)

作成したワークフローでは生成したバッジをリポジトリに 直接 push するため、この設定が必要です。

## 3. ワークフローの設定

リポジトリ直下に`.github/workflows`ディレクトリを作成します。その中に自身の好きなファイル名で良いので YAML ファイルを作成します。

```yaml
name: Kaggle Badges

on:
  push:
    branches:
      - main
  schedule:
    # You can change the cron expression to suit your needs
    - cron: "11 11 1 * *" # 11:11 AM on the 1st of every month
  workflow_dispatch:

jobs:
  create-badges:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install Puppeteer browser
        run: npx puppeteer browsers install chrome@131.0.6778.85

      - name: Use Kaggle Badges Action
        uses: takayoshi-makabe/kaggle-badges@v1.4.0
        with:
          # ex. user_name: spidermandance
          user_name: { Your Kaggle Username }
          # example of using GitHub Secrets
          # user_name: ${{ secrets.KAGGLE_USERNAME }}

      - name: Commit and Push SVG files
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add ./kaggle-badges/* ./kaggle-plates/*
          git commit -m "Add generated SVG files" || echo "No changes to commit"
          git push
```

ここではワークフローの細かな説明は省きますが、`Use Kaggle Badges Action`ステップで作成した GitHub Actions を使用しています。`user_name`に自身の Kaggle のユーザー名を指定することで、そのユーザーのバッジを生成します。
ファイルに直接ユーザー名を書くことに抵抗がある場合は、GitHub Secrets を使用することもできます。（そこまでセキュアな情報でもないと思うので、上の例では直書きしてます。）

- 参考：[Using secrets in workflow](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions)

## 4. ワークフローの実行

上記のワークフローはスケジューリングされているので、指定したタイミングでバッジが生成されます。また、`workflow_dispatch`を指定しているので手動で実行することも可能です。すぐにでもバッジを生成したい場合は、手動で実行してください。

- 参考：[ワークフローの手動実行](https://docs.github.com/ja/actions/using-workflows/manually-running-a-workflow)

## 5. プロフィールにバッジを表示

ワークフローが正常に終了すると、リポジトリ直下に`kaggle-badges`ディレクトリが作成され、その中に各種バッジが生成されています。その URL を使って README などに貼り付けることで、Kaggle のスコアを GitHub のプロフィールに表示することができます。

![images.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/274ee002-b743-e424-2caa-a2bfe3e18c92.png)

```markdown
# Markdown

![](./kaggle-badges/CompetitionsRank/plastic-black.svg)
![](./kaggle-plates/Competitions/white.svg)
```

```html
<!-- HTML -->
<img src="./kaggle-badges/CompetitionsRank/plastic-black.svg" />
<img src="./kaggle-plates/Competitions/white.svg" />
```

# 何をしているのかについて軽く

この GitHub Actions は Puppeteer を使用して Kaggle のユーザーページをスクレイピングし、その情報をもとにバッジを生成しています。
また本機能とは別のテーマですが、 一応 test(jest)に関するワークフローを用意しており、main branch への PR 発行タイミングで test が走ります。GitHub Repository Protection により、main ブランチへの直接 push を禁止しているので、この test に通らないとマージできないようにしています。
あとは`.github/dependabot.yml`を設定しているので、依存パッケージのアップデートをメジャー・マイナー・パッチそれぞれ別の PR に分けて定期的に提案してくれるようにしています。

# 今後

こちらの GitHub Actions はまだ初期バージョンですが、今後もし使っていただくことがありそうならよりスタイリッシュなデザインのバッジも追加したいなと思っています。（JavaScript 周りの知識が浅いので、あまり自信はありませんが、、、）
