---
title: "認証と認可"
---

## 要約

アプリケーションで認証状態を保持する代表的な方法としては以下2つが挙げられ、Next.jsにおいてもこれらを実装することが可能です。

- 保持したい情報をCookieに保持（JWTは必須）
- セッションとしてRedisなどに保持（JWTは任意）

また、代表的な認可チェックには以下2つが考えられます。

- URL認可
- データアクセス認可

これらの認可チェックは両立が可能ですが、URL認可の実装にはNext.jsならではの制約が伴います。

## 背景

Webアプリケーションにおいて、認証と認可は非常にありふれた一般的な要件です。

:::message
認証と認可は混在されがちですが、別物です。これらの違いについて自信がない方は、筆者の[過去記事↗︎](https://zenn.dev/akfm/articles/authentication-with-security)を参照ください。
:::

しかし、Next.jsにおける認証認可の実装には、従来のWebフレームワークとは異なる独自の制約が伴います。

これはNext.jsが、RSCという**自律分散性**と**並行実行性**を重視したアーキテクチャ^[参考: [Reactチームが見てる世界、Reactユーザーが見てる世界↗︎](https://zenn.dev/akfm/articles/react-team-vision)]に基づいて構築されていることや、edgeランタイムとNode.jsランタイムなど**多層の実行環境**を持つといった、従来のWebフレームワークとは異なる特徴を持つことに起因します。

### 並行レンダリングされるページとレイアウト

Next.jsでは、Route間で共通となる[レイアウト↗︎](https://nextjs.org/docs/app/getting-started/layouts-and-pages)を`layout.tsx`などで定義することができます。特定のRoute配下（e.g. `/dashboard`配下など）に対する認可チェックをレイアウト層で一律実装できるのでは、と考える方もいらっしゃると思います。しかし、このような実装は一見期待通りに動いてるように見えても、RSC Payloadなどを通じて情報漏洩などにつながるリスクがあり、避けるべき実装です。

これは、Next.jsの並行実行性に起因する制約です。Next.jsにおいてページとレイアウトは並行にレンダリングされるため、必ずしもレイアウト層に実装した認可チェックがページより先に実行されるとは限りません。そのため、ページ側で認可チェックをしていないと予期せぬデータ漏洩が起きる可能性があります。

これらの詳細な解説については以下の記事が参考になります。

https://zenn.dev/moozaru/articles/0d6c4596425da9

### Server ComponentsでCookie操作は行えない

RSCではデータ取得をServer Components、データ変更をServer Actionsという責務分けがされています。[Server Componentsの純粋性](part_4_pure_server_components)でも述べたように、Server Componentsにおける並行レンダリングやRequest Memoizationは、レンダリングに副作用が含まれないという前提の元に設計されています。

Cookie操作は他のコンポーネントのレンダリングに影響する可能性がある副作用です。そのため、Next.jsにおけるCookie操作である`.set()`や`.delete()`は、Server ActionsかRoute Handler内でのみ行うことができます。

### 制限を伴うmiddleware

Next.jsのmiddlewareはユーザーからのリクエストに対して一律処理を差し込むことができますが、v15.4まではランタイムがedgeに限定されており、Node.js APIが利用できなかったりDB操作系が非推奨など様々な制限が伴います。

:::message
[Next.js v15.5↗︎](https://nextjs.org/blog/next-15-5#nodejs-middleware-stable)でmiddlewareのNode.jsランタイムがStableとなりました。
:::

## 設計・プラクティス

Next.jsにおける認証認可は、上述の制約を踏まえて実装する必要があります。考えるべきポイントは大きく以下の3つです。

- [#認証状態の保持](#認証状態の保持)
- [#URL認可](#URL認可)
- [#データアクセス認可](#データアクセス認可)

### 認証状態の保持

サーバー側で認証状態を参照したい場合、Cookieを利用することが一般的です。認証状態をJWTにして直接Cookieに格納するか、もしくはRedisなどにセッション状態を保持してCookieにはセッションIDを格納するなどの方法が考えられます。

公式ドキュメントに詳細な解説があるので、本書でこれらの詳細は割愛します。

https://nextjs.org/docs/app/guides/authentication#session-management

筆者は、拡張されたOAuthやOIDCを用いることが多く、セッションIDをJWTにしてCookieに格納し、セッション自体はRedisに保持する方法をよく利用します。こうすることで、アクセストークンやIDトークンをブラウザ側に送信せずに済み、セキュリティ性の向上やCookieのサイズを節約などのメリットが得られます。また、Cookieに格納するセッションIDはJWTにすることで、改竄を防止することができます。

:::details GitHub OAuthアプリのサンプル実装
以下はGitHub OAuthアプリとして実装したサンプル実装の一部です。GitHubからリダイレクト後、CSRF攻撃対策のためのstateトークン検証、アクセストークンの取得、セッション保持などを行っています。

https://github.com/AkifumiSato/nextjs-book-oauth-app-example/blob/main/app/api/github/callback/route.ts
:::

### URL認可

URL認可の実装は多くの場合、認証状態や認証情報に基づいて行われます。認可処理を`verifySession()`として共通化した場合、各ページで以下のような実装を行うことになるでしょう。

```tsx :page.tsx
export default async function Page() {
  await verifySession(); // 認可に失敗したら`/login`にリダイレクト

  // ...
}
```

CookieにJWTを格納している場合は、middlewareでJWTの検証を行うことができます。認証状態をJWTに含めている場合は、さらに細かいチェックも可能です。一方、セッションIDのみをJWTに含めるようにしている場合には、IDの有効性やセッション情報の取得にRedisやDB接続が必要になるため、edgeランタイムのmiddlewareで行えるのは[楽観的チェック↗︎](https://nextjs.org/docs/app/guides/authentication#optimistic-checks-with-middleware-optional)に留まるということに注意しましょう。

### データアクセス認可

[Request Memoization](part_1_request_memoization)で述べたように、Next.jsではデータフェッチ層を分離して実装するケースが多々あります。データアクセス認可が必要な場合は、分離したデータフェッチ層で実装しましょう。

例えばVercelのようなSaaSにおいて、有償プランユーザーのみが利用可能なデータアクセスがあった場合、データフェッチ層に以下のような認可チェックを実装すべきでしょう。

```ts
// 🚨`unauthorized()`はv15時点でExperimental
import { unauthorized } from "next/navigation";

export async function fetchPaidOnlyData() {
  if (!(await isPaidUser())) {
    unauthorized();
  }

  // ...
}
```

X（旧Twitter）のようにブロックやミュートなど、きめ細かいアクセス制御（Fine-Grained Access Control）が必要な場合は、バックエンドAPIにアクセス制御を隠蔽する場合もあります。

```ts
// 🚨`forbidden()`はv15時点でExperimental
import { forbidden } from "next/navigation";

export async function fetchPost(postId: string) {
  const res = await fetch(`https://dummyjson.com/posts/${postId}`);
  if (res.status === 401) {
    forbidden();
  }

  return (await res.json()) as Post;
}
```

:::message
[公式ドキュメント↗︎](https://nextjs.org/docs/app/guides/authentication#creating-a-data-access-layer-dal)やVercelの[SaaS参考実装↗︎](https://github.com/vercel/nextjs-subscription-payments)では、認可エラーで`redirect("/login")`のようにリダイレクトするのみのものが多いですが、認可エラー=必ずしもリダイレクトではありません。
:::

:::message
Next.jsはStreamingをサポートしているため、確実にHTTPステータスコードを設定する手段がありません。そのため、上記参考実装の`unauthorized()`や`forbidden()`利用時もHTTPステータスコードが200になる可能性があります。
詳しくは[リクエストの参照とレスポンスの操作](part_5_request_ref)を参照ください。
:::

## トレードオフ

### URL認可の冗長な実装

認証状態に基づくURL認可はありふれた要件ですが、認証状態を確認するのにRedisやDBのデータ参照が必要な場合、edgeランタイムのmiddlewareでは行えません。そのため、前述のように各`page.tsx`で認可チェックを行う必要があり、冗長な実装になります。

```tsx :page.tsx
export default async function Page() {
  await verifySession(); // 認可に失敗したら`/login`にリダイレクト

  // ...
}
```
