---
title: "Next.jsにOAuthクライアントをライブラリなしで実装する"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "oauth"]
published: false
---

[OAuth2.0](https://openid-foundation-japan.github.io/rfc6749.ja.html)は3rd partyアプリケーションがユーザーに代わってリソースサーバーへアクセスすることを可能にする、認可フレームワークです。X（Twitter）やGithub、Facebookなどの著名なOAuthプロバイダーはそれぞれ拡張や制限を儲けることで**認証**にも対応しており、「OAuth認証」という言葉が多く溢れていますが、OAuth自体はあくまで**認可**の仕組みです。これらの違いや注意点については筆者の[過去の記事](https://zenn.dev/akfm/articles/authentication-with-security)を参照いただけたらと思います。

前述のようなOAuthプロバイダーを利用して認証を実装したいことはよくある要件です。筆者はNext.jsを扱うことが多いのですが、Next.jsにおいてOAuthを扱おうと思った時には[NextAuth](https://next-auth.js.org/)を検討される方も多いでしょう。筆者がこのライブラリを試したのはだいぶ前ですが、かなりライトに認証を導入できた印象が記憶に残っています。一方でNextAuthはじめOAuthのライブラリは当然処理を隠蔽するため、「どんな処理をしてるかわからない」「正しい使い方なのかわからない」などの不安を抱く方も多いのではないでしょうか。認証周りはユーザーの個人情報を扱う最も重要な部分であり、これらの不安を解消すべく理解に努めることはとても大切です。

本稿では表題の通りNextAuthなどのライブラリをあえて採用せずスクラッチでOAuthクライアントを[Next.js App Router](https://nextjs.org/docs/app)上に実装することで、これらのライブラリが行ってる処理やOAuthの仕様について、そしてApp Routerにおけるセッション管理について理解を深めることを目指します。

## 実装要件

本稿で実装するアプリケーション概要としては以下の通りです。

- GitHub OAuth(認可コード付与)でアクセストークンの取得を行う
- `state`パラメータを検証しCSRF攻撃対策を行う
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
   - Authorization callback URL: `http://localhost:3000/login/callback`
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

## App Routerにおけるセッション管理

必要なものは揃ったので、実装に移ります。まずはセッション管理を実装し、その後OAuthの仕様に沿ってリダイレクトやAPIリクエストなどを実装していきます。

### セッションの設計

セッションには認証状態とGitHubのアクセストークンを保存しておきたいので、Redisに保存する構造はタグ付きunionで定義すると以下のような型になります。ファイルは`app/lib/session.ts`とします。

```ts
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

このことから、上記に定義した`RedisSession`を読み取り専用で参照する場合と、変更可能なセッションとして扱う2つが考えられます。前者はReadOnlyな`RedisSession`で十分ですが、後者は`MutableSession`クラスとしてセッション操作時のメソッドを定義することにします。

```ts
class MutableSession {
  private readonly redisSession: RedisSession;

  constructor(redisSession: RedisSession) {
    this.redisSession = redisSession;
  }

  get currentUser() {
    return this.redisSession.currentUser;
  }

  private async save(): Promise<void> { /* ... */ }
}
```

この`MutableSession`に必要に応じて変更の振る舞いを追加していきます。これらのクラスや構造を取得する関数として、以下を定義します。

```ts
export async function getMutableSession(): Promise<MutableSession> { /* ... */ }
export async function getReadonlySession(): Promise<Readonly<RedisSession>> { /* ... */ }
```

`page.tsx`やServer Actionそれぞれ上記関数を呼び出してセッションを扱うものとします。

さて、`MutableSession`の`save`が未実装だったので、Redisへの保存とNext.jsの`cookies`でセッションIDの設定を行います。

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

これでセッションを利用する準備は整いました。以降はOAuthのフローを実装しながら`MutableSession`の振る舞いを実装していきます。

## OAuthの認可コード付与の実装

OAuth2.0ではアクセストークンを取得する方法としていくつかのフローを定義しています。最も標準的なのは[認可コード付与](https://tex2e.github.io/rfc-translater/html/rfc6749.html#4-1--Authorization-Code-Grant)という手法で、Github OAuthでもこれをサポートしており、Webアプリケーションでは通常このフローを採用します。

:::message
GitHubでは[デバイス認証付与](https://tex2e.github.io/rfc-translater/html/rfc8628.html)もサポートしていますが、これはブラウザが利用できないCLIやツールなどでの採用を想定しています。
:::

ここからは以下のGitHun公式ドキュメントに沿って、認可コード付与の実装を行います。

https://docs.github.com/ja/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps#web-application-flow

### 1. ユーザーの GitHub ID を要求する

まずはGitHubの認可ページにリダイレクトします。リダイレクト前に、`state`パラメータに付与するCSRFトークンを発行しセッションに保存する必要があります。そのため、Next.jsでの実装はServer Actionsでセッション操作後にGitHubへリダイレクトすることになります。

```tsx
// app/page.tsx
import { login } from "./action";

export default function Page() {
  return (
    <>
      <h1>Hello, Github OAuth App!</h1>
      <form action={login}>
        <button type="submit">Github OAuth</button>
      </form>
    </>
  );
}

// app/action.ts
"use server";

import { redirect } from "next/navigation";
import { getMutableSession } from "./lib/session";

export async function login() {
  const mutableSession = await getMutableSession();
  const state = await mutableSession.preLogin();

  redirect(
    `https://github.com/login/oauth/authorize?scope=user:email&client_id=${process.env.GITHUB_CLIENT_ID}&state=${state}`,
  );
}
```

`state`以外にもパラメータに`client_id`と`scope`を指定しています。`client_id`はGitHubのOAuthアプリケーションの設定で取得したものです。`scope`はGitHubの認可ページで要求する権限を指定します。ここでは`user:email`を指定していますが、他にも[様々なスコープ](https://docs.github.com/ja/apps/oauth-apps/building-oauth-apps/scopes-for-oauth-apps#available-scopes)があります。

`mutableSession.preLogin()`はCSRFトークンを発行し、セッションに保存するメソッドです。このメソッドを`MutableSession`に実装します。

```tsx
// app/lib/session.ts
class MutableSession {
  // ...
  async preLogin() {
    const state = uuid();
    this.redisSession.currentUser = { isLogin: false, state };
    await this.save();

    return state;
  }
  // ...
}
```

これでGitHubの認可ページにリダイレクトする準備が整いました。

### 2. GitHub によってユーザーが元のサイトにリダイレクトされる

GitHubの認可ページでユーザーが認可を行うと、指定したURLにリダイレクトされます。このリダイレクト先のURLはOAuthアプリケーションの設定で指定したものです。リダイレクト時にはGETパラメータで`state`と`code`が渡されます。`state`はセッションのCSRFトークンと照合し、異なる値であれば処理を中断しなければなりません。

これは、CSRF攻撃を防ぐための措置で、**GitHubへ認証しに行った人とGitHubから認証して帰ってきた人が同一である**ことを確認するためのものです。これらが異なる場合、悪意ある攻撃者が他人を自分のアカウントで認証させ用途している可能性があります。例えば筆者がGitHubで認証しリダイレクトされるURLが発行された段階でリクエストを停止し、読者であるあなたに送りつけたとします。あなたがそのURLをクリックすると、筆者のアカウントで他アプリケーションにログインした状態になってしまいます。このままあなたが気づかず、未入力になっていた個人情報を入力するとどうでしょう？筆者は同じアカウントでログイン可能なので、個人情報の奪取に成功してしまいます。これを防ぐために、`state`パラメータによるCSRFトークンの検証が必要なのです。

https://qiita.com/ist-n-m/items/67a5a0fb4f50ac1e30c1#oauth20-%E3%81%AE-csrfcross-stie-request-forgery

:::message
実際には`code`の有効期限が10分なので、攻撃リスクは低いかもしれませんが、被害が出てからでは遅いので`state`パラメータの検証は行うようにしましょう。
:::

さて、前述の設定で`http://localhost:3000/login/callback` が指定しているので、ここにリダイレクト後の処理を実装します。処理の流れは以下のようになります。

1. セッションのCSRFトークンとリダイレクト時の`state`パラメータを照合
2. `code`パラメータを取得
3. `code`パラメータを使ってGitHubにアクセストークンを要求
4. アクセストークンを取得
5. セッションにアクセストークンを保存
6. `/user`へリダイレクト

`GITHUB_CLIENT_ID`と`GITHUB_CLIENT_SECRET`はGitHubのOAuthアプリケーションの設定で取得したものです。

```ts
// app/(auth)/login/callback/route.ts
import { RedirectType, redirect } from "next/navigation";
import { NextRequest } from "next/server";
import { getMutableSession } from "../../../lib/session";

type GithubAccessTokenResponse = {
  access_token: string;
  token_type: string;
  scope: string;
};

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const mutableSession = await getMutableSession();
  if (mutableSession.currentUser.isLogin === true) {
    throw new Error("Already login.");
  }

  // check state(csrf token)
  const urlState = searchParams.get("state");
  if (mutableSession.currentUser.state !== urlState) {
    console.error("CSRF Token", mutableSession.currentUser.state, urlState);
    throw new Error("CSRF Token not equaled.");
  }

  const code = searchParams.get("code");
  const { GITHUB_CLIENT_ID, GITHUB_CLIENT_SECRET } = process.env;
  if (GITHUB_CLIENT_ID === undefined || GITHUB_CLIENT_SECRET === undefined) {
    throw new Error("GITHUB_CLIENT_ID or GITHUB_CLIENT_SECRET is not defined");
  }

  const githubTokenResponse: GithubAccessTokenResponse = await fetch(
    `https://github.com/login/oauth/access_token?client_id=${GITHUB_CLIENT_ID}&client_secret=${GITHUB_CLIENT_SECRET}&code=${code}`,
    {
      method: "GET",
      headers: {
        Accept: " application/json",
      },
    },
  ).then((res) => {
    if (!res.ok) throw new Error("failed to get access token");
    return res.json();
  });

  await mutableSession.onLogin(githubTokenResponse.access_token);

  redirect("/user", RedirectType.replace);
}

// app/lib/session.ts
class MutableSession {
  // ...
  async onLogin(accessToken: string) {
    this.redisSession.currentUser = { isLogin: true, accessToken };
    await this.save();
  }
  // ...
}
```

これでアクセストークンを取得することに成功しました。`/user`ページで実際にGitHub APIを叩いてユーザー情報を取得してみます。

### 3. アクセストークンを使ってAPIにアクセスする

`/user`では、ログインを必須とするページな想定として`session.currentUser.isLogin`をチェックし、ログインしていない場合は`NotLogin`コンポーネントを返すようにします。

認証済みの場合、アクセストークンを保持してるのでこれを使いGitHub APIからユーザー情報を取得します。リクエスト時には`Authorization`ヘッダーに`Bearer ${session.currentUser.accessToken}`を付与する必要があります。

```tsx
// app/(auth)/user/page.tsx
import { getReadonlySession } from "../../lib/session";
import { GithubUser, NotLogin } from "./presentational";

// Partial type
export type GithubUserResponse = {
  id: number;
  name: string;
  email: string;
};

export default async function Page() {
  const session = await getReadonlySession();
  if (!session.currentUser.isLogin) {
    return <NotLogin />;
  }

  const githubUser: GithubUserResponse = await fetch(
    "https://api.github.com/user",
    {
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${session.currentUser.accessToken}`,
      },
    },
  ).then(async (res) => {
    if (!res.ok) {
      console.error(res.status, await res.json());
      throw new Error("failed to get github user");
    }
    return res.json();
  });

  return <GithubUser githubUser={githubUser} />;
}
```

GitHub OAuthの認可コード付与の実装はこれで以上です。アクセストークンを取得し、GitHub APIを利用することができるようになりました。

## より深く理解するために

自分でプロバイダーのドキュメントを読みながらOAuthクライアントを実装してみると、悪意ある攻撃やそれらに対する保護方法など、多くの学びが得られます。OAuth2.0やOpen ID Connectの仕様を明記してるRFCを読むとさらにより深い理解を得られるので、業務でこれらを利用すると言う方はぜひ一度RFCも読んでみることをお勧めします。

### 余談: 単体テストの実装

Todo: 単体テストの導入〜実装まで解説

本稿の参考実装で登場するテストコードは[Vitest](https://vitest.dev/)で実装されているので、こちらの導入もおすすめします。

```shell
$ pnpm add -D vitest @vitejs/plugin-react ioredis-mock msw
```

- vitest
    - [/vitest.config.mts](https://github.com/AkifumiSato/next-oauth-pure-impl-example/blob/main/vitest.config.mts)
    - [/vitest.setup.ts](https://github.com/AkifumiSato/next-oauth-pure-impl-example/blob/main/vitest.setup.ts)
- msw
    - [/app/mocks.ts](https://github.com/AkifumiSato/next-oauth-pure-impl-example/blob/main/app/mocks.ts)
