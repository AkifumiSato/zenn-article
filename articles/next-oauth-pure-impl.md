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

本稿では表題の通りNextAuthなどのライブラリをあえて採用せずスクラッチでOAuthクライアントを[Next.js App Router](https://nextjs.org/docs/app)上に実装することで、これらのライブラリが行ってる処理やOAuthの仕様について、そしてApp Routerにおけるセッション管理について理解を深めることを目指します。

## 実装要件

本稿で実装するアプリケーションの要件としては以下の通りです。

- GitHub OAuth(Authorization Code Grant)でアクセストークンの取得を行う
- 取得したアクセストークンはサーバー側セッションに保存する
- セッション管理はRedisで行う

:::message
GihHub OAuthを選んだのは、単に多くの開発者がアカウントを持ってると思ったからです。<br />基本的な処理の流れは変わらないので、他のプロバイダーでも構いません。
:::

## 参考実装

実装の全量を記載するとわかりにくいため一部実装を省略記載してる部分もあります。実装の全量については以下のリポジトリをご参照ください。

https://github.com/AkifumiSato/next-oauth-pure-impl-example

## 設定・環境構築

まずは先に設定と環境構築です。

### GitHubにOAuthアプリケーションを設定

1. GitHubにログイン
2. [OAuth Apps](https://github.com/settings/developers)にアクセス
3. 「New OAuth App」をクリック
4. 必要な情報を入力
   - Application name: に任意の名前 
   - Homepage URL: `http://localhost:3000` 
   - Authorization callback URL: `http://localhost:3000/api/auth/callback/github`
5. 「Register application」をクリック
6. Client IDとClient Secretを控えておく

![GitHubにOAuthアプリケーションを設定](/images/next-oauth-pure-impl/github_register_app.png)

:::message alert
Client Secretは遷移すると表示されなくなるのでご注意ください。
再発行も可能なので、表示されなくなってしまった時は再発行しましょう。
:::

### Next.js App Routerプロジェクトの作成

Next.jsのプロジェクトを作成します。できるだけシンプルな雛形を使いたいので`--example hello-world`をつけています。また、筆者はpnpm推しなので`--use-pnpm`をつけています。

```shell
$ pnpm create next-app --use-pnpm --example hello-world
```

### Redisをdocker-composeで起動

作業PCにDockerがインストールされていることを前提とします。`docker-compose.yml`を作成し、ローカル環境でRedisを起動します。

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

```shell
$ docker-compose up
```

Redis接続のために[ioredis](https://www.npmjs.com/package/ioredis)もインストールしておきます。

```shell
$ pnpm add ioredis
```

### テスティングツール

本稿の参考実装で登場するテストコードは[Vitest](https://vitest.dev/)で実装されているので、こちらの導入もおすすめします。

```shell
$ pnpm add -D vitest @vitejs/plugin-react ioredis-mock msw
```

各種設定は以下の参考実装をご参照ください。

TBW: 各設定や導入について詳細記述するか検討

- vitest
  - [/vitest.config.mts](https://github.com/AkifumiSato/next-oauth-pure-impl-example/blob/main/vitest.config.mts)
  - [/vitest.setup.ts](https://github.com/AkifumiSato/next-oauth-pure-impl-example/blob/main/vitest.setup.ts)
- msw
  - [/app/mocks.ts](https://github.com/AkifumiSato/next-oauth-pure-impl-example/blob/main/app/mocks.ts)
- Biome
  - [/biome.json](https://github.com/AkifumiSato/next-oauth-pure-impl-example/blob/main/biome.json)

## App Routerにおけるセッション管理

必要なものは揃ったので、実装に移ります。まずはセッション管理を実装し、その後は公式ドキュメントにリダイレクトやAPIリクエストなどを実装していきます。

### Sessionの設計

まずはセッションとしてRedisに永続化する構造を設計します。セッションには今回、認証状態とGitHubのアクセストークンを保存しておきたいので、以下のようなtypeになります。

```ts
// app/session.ts
type RedisSession = {
  currentUser:
    | {
    isLogin: false;
  }
    | {
    isLogin: true;
    accessToken: string;
  };
};
```

セッションはCookieにセッションIDを保存することで実現するわけですが、App RouterにおいてCookie操作はServer ActionやRoute Handlerに限られます。

https://nextjs.org/docs/app/api-reference/functions/cookies#cookiessetname-value-options

このことから、上記に定義した`RedisSession`を読み取り専用で参照する場合と、変更可能なセッションとして扱う2つが考えられます。前者は`RedisSession`のままで十分ですが、後者は`MutableSession`クラスとしてセッション操作時の振る舞いを定義します。

```ts
class MutableSession {
  private readonly redisSession: RedisSession;

  constructor(redisSession: RedisSession) {
    this.redisSession = redisSession;
  }

  get currentUser() {
    return this.redisSession.currentUser;
  }

  async preLogin() { /* ... */ }

  async onLogin(accessToken: string) { /* ... */ }

  async onLogout() { /* ... */ }

  private async save(): Promise<void> { /* ... */ }
}
```

各メソッドの実装や詳細については後述します。この時点ではメソッド名だけ定義しておきます。

これらのクラスや構造を取得する関数として、以下を定義します。

```ts
export async function getMutableSession(): Promise<MutableSession> { /* ... */ }
export async function getReadonlySession(): Promise<Readonly<RedisSession>> { /* ... */ }
```

`page.tsx`やServer Actionそれぞれ上記関数を呼び出してセッションを扱うものとします。

さて、現段階で実装ではまだ`preLogin`や`onLogin`について実装できる部分はないですが、できる部分のみ実装しましょう。`MutableSession`の`save`でRedisへの保存とNext.jsの`cookies`でセッションIDの設定を行います。

```ts
import Redis from "ioredis";
import { cookies } from "next/headers";
import { v4 as uuid } from "uuid";

const SESSION_COOKIE_NAME = "sessionId";

// ...

class MutableSession {
  // ...

  private async save(): Promise<void> {
    const sessionIdFromCookie = cookies().get(SESSION_COOKIE_NAME)?.value;
    let sessionId: string;
    if (sessionIdFromCookie) {
      sessionId = sessionIdFromCookie;
    } else {
      sessionId = uuid();
      cookies().set(SESSION_COOKIE_NAME, sessionId, {
        httpOnly: true,
        // localhost以外で動作させる場合はsecure: trueを有効にする
        // secure: true,
      });
    }
    await redisStore.set(sessionId, JSON.stringify(this.values));
  }

  // ...
}
```

`MutableSession`は初期値をコンスラクタに取るので、`getMutableSession`でCookieやRedisを参照して初期値を取得する`loadPersistedSession`関数を定義します。これを利用して、`getMutableSession`や`getReadonlySession`を実装します。

```ts
async function loadPersistedSession(): Promise<RedisSession> {
  const sessionIdFromCookie = cookies().get(SESSION_COOKIE_NAME)?.value;
  const session = sessionIdFromCookie
    ? await redisStore.get(sessionIdFromCookie)
    : null;
  if (session) {
    return JSON.parse(session) as RedisSession;
  }
  return { currentUser: { isLogin: false } };
}

// use only in actions/route handlers
export async function getMutableSession(): Promise<MutableSession> {
  return new MutableSession(await loadPersistedSession());
}

// readonly session
export async function getReadonlySession(): Promise<
  Readonly<RedisSession>
> {
  return await loadPersistedSession();
}
```

以降はOAuthのフローを実装しながら`MutableSession`の振る舞いを実装していきます。

## OAuthの認可コードフローの実装

TBW

## 構成

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
