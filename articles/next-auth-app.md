---
title: "Next.jsと認証の基礎"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

認証や認可、Next.js で実装する時の構成とかについて時間が経つとすぐ忘れてしまうので、記事で残しておこうかと思います。

## 認証と認可

**認証**（Authentication）と**認可**（Authorization）は別なものです。OAuth は 3rd party 向けの認可の仕組みを定義したものであって、認証の仕組みではないのでこの差を理解しないまま流用すると誤った認証や攻撃対象となってしまうリスクを伴います。

:::details 参考記事

- [OAuth 認証とは何か?なぜダメなのか](https://ritou.hatenablog.com/entry/2020/12/01/000000)
- [単なる OAuth 2.0 を認証に使うと、車が通れるほどのどでかいセキュリティー・ホールができる](https://www.sakimura.org/2012/02/1487/)
- [OAuth 2.0 に潜む「5 つの脆弱性」と解決法](https://atmarkit.itmedia.co.jp/ait/articles/1710/24/news011_2.html)
  :::

### OAuth2

### OAuth 認証の脆弱性

#### CSRF

#### リプレイ攻撃

<!-- 以下前に書いた際の残り
題材的には筆者が以前から趣味で作ってる~~永遠に完成しない~~Web アプリ（Next.js+Nest.js） の構成の話を中心とします。

### 作ってるもの

作ってるものはざっくりこんな感じです。
ドメインとかも取ってあるけど、正直勉強がてら作ってるのでこのまま永遠に完成しないか完全作り直すかもなので詳細は省略。

- Google 認証付き Web アプリケーション
- Vercel+Next と AWS copilot with Fargate+Nest でデプロイしたい

ISR したかったのとデプロイやセキュリティ周りとか考える量も減るので、フロントは Vercel にしました。API は永続化できないことにはしょうがないので AWS copilot を使って Fargate で構成してます。
ということでフロント側は Vercel、バックエンドは AWS Fargate で構築してる API という構成になっています。
 -->

## 構成メモ

- introduction
  - Next と認証のメモだよ
- 認証と認可
  - 認証と認可
  - OAuth2
  - OAuth 認証の脆弱性
    - CSRF
    - リプレイ攻撃
- Open ID Connect
  - Open ID Connect とは
  - ID トークンから取得できるもの
  - nonce の検証
  - cookie に格納する際の設定
    - http only
    - secure
    - same site
- 実際の処理
  - 処理フロー
    - （図）
    - API へアクセス
    - Cookie 付与
    - 次回からログイン情報取れる
  - cookie
    - http only
    - same site
    - secure
