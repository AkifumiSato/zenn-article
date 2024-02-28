---
title: "Next.jsにOAuthクライアントをライブラリなしで実装する"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "oauth"]
published: false
---

[OAuth2.0](https://openid-foundation-japan.github.io/rfc6749.ja.html)は3rd partyアプリケーションがユーザーに代わってリソースサーバーへアクセスすることを可能にする、認可フレームワークです。X（Twitter）やGithub、Facebookなどの著名なOAuthプロバイダーはそれぞれ拡張や制限を儲けることで**認証**にも対応しており、「OAuth認証」という言葉が多く溢れています。

:::message
OAuth自体はあくまで**認可**の仕組みです。これらの違いや注意点については筆者の[過去の記事](https://zenn.dev/akfm/articles/authentication-with-security)を参照いただけたらと思います。
:::

とはいえ、前述のようなOAuthプロバイダーを利用して認証を実装したいことはよくある要件です。筆者はNext.jsを扱うことが多いのですが、Next.jsにおいてOAuthを扱おうと思った時には[NextAuth](https://next-auth.js.org/)を検討される方も多いでしょう。筆者がこのライブラリを試したのはだいぶ前ですが、かなりライトに認証を導入できた印象が記憶に残っています。一方でNextAuthはじめOAuthのライブラリは当然処理を隠蔽するため、「どんな処理をしてるかわからない」「正しい使い方なのかわからない」などの不安を抱く方も多いのではないでしょうか。認証周りはユーザーの個人情報を扱う最も重要な部分であり、これらの不安を解消すべく理解に努めることはとても大切です。

本稿では表題の通りNextAuthなどのライブラリをあえて採用せずスクラッチでOAuthクライアントを[Next.js App Router](https://nextjs.org/docs/app)上に実装することで、これらのライブラリが行ってる処理やOAuthの仕様について理解、そしてApp Routerにおけるセッション管理について理解を深めることを目指します。

## 実装要件

- GitHub OAuth(Authorization Code Grant)でアクセストークンの取得を行う
- 取得したアクセストークンはサーバー側セッションに保存する
- セッション管理はRedisで行う

:::message
GihHub OAuthを選んだのは、単に多くの開発者がアカウントを持ってると思ったからです。<br />基本的な処理の流れは変わらないので、他のプロバイダーでも構いません。
:::

## 事前準備

### GitHubにOAuthアプリケーションを設定

1. GitHubにログイン
2. [OAuth Apps](https://github.com/settings/developers)
3. 「New OAuth App」をクリック
4. 必要な情報を入力
   - Application name: に任意の名前 
   - Homepage URL: `http://localhost:3000` 
   - Authorization callback URL: `http://localhost:3000/api/auth/callback/github`

![GitHubにOAuthアプリケーションを設定](/images/next-oauth-pure-impl/github_register_app.png)

### Next.js App Routerプロジェクトの作成

Next.jsのプロジェクトを作成します。できるだけシンプルな雛形を使いたいので`--example hello-world`をつけています。また、筆者はpnpm推しなので`--use-pnpm`をつけています。

```shell
$ pnpm create next-app --use-pnpm --example hello-world
```

### Redisをdocker-composeで起動

作業PCにDockerがインストールされていることを前提とします。`docker-compose.yml`を作成し、`docker-compose up`でローカル環境でRedisを起動します。

```yml
# docker-compose.yml
services:
  redis:
    image: redis
    ports:
      - 6379:6379
    expose:
      - 6379
    container_name: next_oauth_pure_impl_example_redis
    volumes:
      - next-oauth-pure-impl-example-redis:/data
    restart: always
volumes:
  next-oauth-pure-impl-example-redis:
    driver: local
```

## OAuthクライアントの実装

## 構成

- App Routerにおけるセッション管理
  - todo: ReadonlySessionの実装
- OAuthの認可コードフローの実装
  - OAuthプロバイダーに遷移する前に、セッションにCSRF tokenを保存
  - OAuthプロバイダーにstateパラメータ付きでリダイレクト
  - OAuthプロバイダー側で認証
  - callbackのURLでstateの検証
  - token取得
  - userの取得
  - 成功
- 次のステップ
  - RFCを読んでみる
