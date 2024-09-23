---
title: "認証と認可"
---

## 要約

アプリケーションで認証状態を保持する代表的な方法としては以下2つが挙げられ、App Routerにおいてもこれらを実装することが可能です。

- 保持したい情報をCookieに保持（JWTは必須）
- セッションとしてRedisなどに保持（JWTは任意）

また、アプリケーションで保持した認証状態に基づく認可チェックには、以下2つの方法が考えられます。

- URLに対する認可チェック
- データリソースに対する認可チェック

これらは両立が可能ですが、前者の実装にはApp Routerならではのいくつかの制約が伴います。

## 背景

Webアプリケーションにおいて、認証と認可は非常にありふれた一般的な要件です。

:::message
認証と認可は混在されがちですが、別物です。これらの違いについて自信がない方は、筆者の[過去記事](https://zenn.dev/akfm/articles/authentication-with-security)を参照ください。
:::

しかし、App Routerにおける認証認可の実装には、従来のWebフレームワークとは異なる独自の制約が伴います。

これはApp Routerが、React Server Componentsという**自律分散性**と**並行実行性**を重視したアーキテクチャに基づいて構築されていることや、edgeランタイムとNode.jsランタイムなど**多層の実行環境**を持つといった、従来のWebフレームワークとは異なる特徴を持つことに起因します。

### 並行レンダリングされるページとレイアウト

App Routerでは、Route間で共通となる[レイアウト](https://nextjs.org/docs/app/building-your-application/routing/layouts-and-templates)を`layout.tsx`などで定義することができます。特定のRoute配下に対する認可チェックをレイアウト層で一律実装できるのでは、と考える方もいらっしゃると思います。しかし、このような実装は一見期待通りに動いてるように見えても、RSC Payloadなどを通じて情報漏洩などにつながるリスクがあり、避けるべき実装です。

これは、App Routerの並行実行性に起因する制約です。App Routerにおいてページとレイアウトは並行にレンダリングされるため、必ずしもレイアウト層の認可チェックがページより先に実行されるとは限りません。意図した仕様なのかは不明ですが、現状だとページの方が先にレンダリングされ始めるようです。そのため、ページ側で認可チェックをしていないと予期せぬデータ漏洩が起きる可能性があります。

これらの詳細な解説については以下の記事が参考になります。

https://zenn.dev/moozaru/articles/0d6c4596425da9

### Server ComponentsでCookie操作は行えない

React Server Componentsでは、データ取得をServer Components・データ変更をServer Actionsという責務分けがされています。Server Componentsにおける並行レンダリングやRequest Memoizationは、レンダリング中にデータ操作が起きえない前提の元設計されています。

Cookie操作は他のコンポーネントのレンダリングに影響する可能性がある、一種のデータ変更です。そのため、App RouterにおけるCookie操作である`cookies().set()`や`cookies().delete()`は、Server ActionsかRoute Handler内でのみ行うことができます。

### 制限を伴うmiddleware

Next.jsのmiddlewareは、ユーザーからのリクエストに対して一律処理を差し込むことができますが、middlewareは本書執筆現在のv14以下ではランタイムがedgeに限定されており、Node APIが利用できなかったりDB操作系が非推奨など、様々な制限が伴います。

将来的にはNode.jsがランタイムとして選択できるようになる可能性はありますが、現状議論中の段階です。

https://github.com/vercel/next.js/discussions/46722#discussioncomment-10262088

## 設計・プラクティス

App Routerにおける認証認可の実装には、上述の制約を踏まえて実装する必要があります。考えるべきポイントは大きく以下の3つです。

- [認証状態の保持](#認証状態の保持)
- [URL認可](#URL認可)
- [データアクセス認可](#データアクセス認可)

### 認証状態の保持

サーバー側で認証状態を参照したい場合はCookieを利用することが一般的です。認証状態をJWTにして直接Cookieに格納するか、もしくはRedisなどにセッション状態を保持してCookieにはセッションIDを格納するなどの方法が考えられます。

公式ドキュメントに詳細な解説があるので、本書では詳細は割愛します。

https://nextjs.org/docs/app/building-your-application/authentication#session-management

筆者は認証拡張されたOAuthやOIDCを用いることが多く、セッションIDをJWTにしてCookieに格納しつつセッション自体はRedisに保持する方法をよく利用します。こうすることで、アクセストークンやIDトークンをブラウザ側に送信せず、Cookieのサイズを節約し、JWTにより改竄を防止することができます。

:::details GitHub OAuthアプリのサンプル実装
以下はGitHub OAuthアプリとして実装したサンプル実装の一部です。GitHubからリダイレクト後、stateトークンの検証、アクセストークンの取得、セッション保持を行っています。

https://github.com/AkifumiSato/nextjs-book-oauth-app-example/blob/main/app/api/github/callback/route.ts
:::

### URL認可

URL認可の実装は多くの場合、認証状態や認証情報に基づいて行われます。App Routerにおいては前述のようにmiddlewareがedgeランタイムでNode.js APIが利用できないため、JWTの検証のみならmiddlewareで行うことが可能です。

RedisやDBのデータ参照が必要な場合には、各ページで認可のチェックを行う必要があります。認可処理を`verifySession()`として共通化した場合、各ページで以下のような実装を行うことになるでしょう。

```tsx
export default async function Page() {
  await verifySession(); // 認可に失敗したら`/login`にリダイレクト

  // ...
}
```

このようにデータ参照が必要な場合でもCookieに格納する情報をJWTにしている場合には、[楽観的チェック](https://nextjs.org/docs/app/building-your-application/authentication#optimistic-checks-with-middleware-optional)としてmiddlewareでJWT検証を行うことができます。

### データアクセス認可

- データアクセス層での認可チェックはFGAC（Fine-Grained Access Control）なUIと相性が良い
- 具体的には「アクセス権限がありません」と表示するようなUIなど
- Vercelのサンプルはデータアクセス層での認可チェックに失敗するとログイン画面に飛ばすようなものもあるが、無限ログインのループになる可能性もあるので注意が必要

## トレードオフ

### URL認可の冗長な実装

- RedisやDBのデータ参照が必要な場合、実装が非常に冗長になる
- これに対する回避策として検討されてるのが、middlewareのNode.jsランタイム対応である
- Vercelのインフラを大きく変更しなければならないためなのか、要望に対しNext.jsコアチームの動きは重いようにも感じる。今後に期待
