---
title: BackgroundMusicとLadioCastを使ってMacで自分の音声とPC音源の両方をGoogle Meetにて共有する (随時更新予定)
tags:
  - BlackHole
  - LadioCast
  - BackgroundMusic
private: false
updated_at: '2025-06-06T14:53:39+09:00'
id: 67613157f7985225cbde
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 16baee61b1d8bd4aac5a
agreed_posting_campaign_term: true
---

# 記事のモチベーション

ミーティングにて自分の音声＋ PC の音声を共有する方法について、いろいろな方法はあると思いますが、最もシンプルな方法についてまとめていきます。他人に向けた記事というより備忘録の意味合いが強いです。環境は以下の通りです

機種：MacBook Air (M1, 2020)
OS：macOS Sonoma (14.5)

# 事前準備

BackgroundMusic のインストール：[GitHub](https://github.com/kyleneideck/BackgroundMusic)
BlackHole のインストール：[Homebrew](https://formulae.brew.sh/cask/blackhole-16ch)
LadioCast のインストール：[Apple Store](https://apps.apple.com/jp/app/ladiocast/id411213048?mt=12)

# 手順

## 1. BackgroundMusic

BackgroundMusic は、Mac の音声出力を管理するアプリです。特に初期設定はいじっていませんが、Output Device は `Macのスピーカー` に設定しておきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/31663c6c-355b-3fb8-2c4d-06d560c246bb.png" width="30%">

また Mac 側のサウンド設定について、Output を BackgroundMusic に設定しておきます。こうすることで、PC の音声を BackgroundMusic を通して出力することができます。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/d3c1eac2-003c-8a39-e7a0-9958eb92cba4.png" width="50%">

Windows と異なり Mac の場合 PC の音声を共有することは結構難易度が高いです、、、。しかし、BackgroundMusic を使えば、Spotfy や YouTube などの音声をまとめて扱うことができるので非常に便利です。OBS を使った配信などでも活用できると思います。

## 2. BlackHole

インストールをすれば、システム環境設定のサウンドから BlackHole が選択できるようになります。特に設定などは必要ありません。BlackHole は、Mac の音声を仮想的にルーティングするためのアプリです。

## 3. LadioCast

ここの設定が設定が一番のポイントです。自分は以下のように設定することで一応 Google Meet での音声共有ができました。（多分色々やり方もあるしベストプラクティスではないかもですが）

入力１：Mac のマイク（自分の声）
入力２：Background Music（PC の音声）
出力 メイン：BlackHole 16ch

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/9c2dbfd4-6c3d-1ad1-ca28-d036c7099080.png" width="50%">

## 4. Google Meet

音声のインプットを BlackHole 16ch に設定することで、Google Meet で自分の音声と PC の音声を共有することができます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/c17ac3a2-55ae-654d-47b6-e1f16cc29660.png" width="40%">

音声のアウトプットを Mac のスピーカーに設定することで、ミーティング相手の声を聞くことができます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/9004b9ca-7fab-e0a6-535e-a367eca48968.png" width="40%">

なお、自分は BGM として Spotify を使うことが多いですが、自分の話している最終に音楽の音量が大きいと結構気になるので、Spotify の音量を下げつつ、他で調整をしています。

![spotify-volume.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/323251/9c3df09c-3f50-085d-2df1-211c4dff7112.png)

一点補足として、Google Meet 側でノイズキャンセリング機能が ON になっている場合、人の音声以外の音を消してしまうことがあります。（これに気づかずめちゃくちゃ時間を溶かしました。。。）
その場合は、Google Meet の設定からノイズキャンセリングを OFF にしてください。

# やろうとして断念したこと

OBS と BackGroundMusic を使えば比較的簡単に、自分の声と PC の音声をミックスする形で配信や録音が可能です。しかもスライドや画面共有も凝った形式で行えます。
ただ OBS の画面も音声も両方合わせて Google Meet に共有するのは比較的難易度が高そうで、NDI を使えばいけるみたいな情報を得たのですが、この辺り Mac と Windows での違いもあり今のところ断念しています。
一応、今も音声はこの形式で声+PC 音声を共有しつつ、OBS の画面共有を Google Meet でやればそれっぽいことは可能です。（パフォーマンス的なことや設定の多さはさておき）
