---
title: "リクエストの参照とレスポンスの操作"
---

## 要約

Server ComponentsやServer Functionsでは他フレームワークにあるようなリクエストオブジェクト(`req`)やレスポンスオブジェクト(`res`)を参照することはできません。代わりに必要な情報を参照するためのAPIが提供されています。

## 背景

Pages Routerなど従来のWebフレームーワークでは、リクエストオブジェクト(`req`)やレスポンスオブジェクト(`res`)を参照することで様々な情報にアクセスしたり、レスポンスをカスタマイズするような設計が広く使われてきました。

```tsx
export const getServerSideProps = (async ({ req, res }) => {
  // リクエスト情報から`sessionId`というcookie情報を取得
  const sessionId = req.cookies.sessionId;

  // レスポンスヘッダーに`Cache-Control`を設定
  res.setHeader(
    "Cache-Control",
    "public, s-maxage=10, stale-while-revalidate=59",
  );

  // ...

  return { props };
}) satisfies GetServerSideProps<Props>;
```

しかし、Server ComponentsやServer Functionsではこれらのオブジェクトを参照することはできません。

## 設計・プラクティス

Next.jsではリクエストやレスポンスオブジェクトを提供する代わりに、必要な情報を参照するためのAPIが提供されています。

:::message
Server Componentsでリクエスト時の情報を参照する関数の一部は[Dynamic APIs↗︎](https://nextjs.org/docs/app/guides/caching#dynamic-apis)と呼ばれ、これらを利用するとRoute全体が[Dynamic Rendering↗︎](https://nextjs.org/docs/app/guides/caching#static-and-dynamic-rendering)となります。
:::

:::message
Next.jsは内部処理の都合で特殊なエラーを`throw`することがあります。そのため、下記のAPIに対し`try {} catch {}`するとNext.jsの動作に影響する可能性があります。
詳しくは[`unstable_rethrow()`↗︎](https://nextjs.org/docs/app/api-reference/functions/unstable_rethrow)を参照ください。
:::

### URL情報の参照

#### `params` props

Dynamic RoutesのURLパスの情報は[`params` props↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/page#params-optional)で提供されます。以下は`/posts/[slug]`と`/posts/[slug]/comments/[commentId]`というルーティングがあった場合の`params`の例です。

| URL                        | `params` props                       |
| -------------------------- | ------------------------------------ |
| `/posts/hoge`              | `{ slug: "hoge" }`                   |
| `/posts/hoge/comments/111` | `{ slug: "hoge", commentId: "111" }` |

```tsx
export default async function Page({
  params,
}: {
  params: Promise<{
    slug: string;
    commentId: string;
  }>;
}) {
  const { slug, commentId } = await params;
  // ...
}
```

#### `useParams()`

[`useParams()`↗︎](https://nextjs.org/docs/app/api-reference/functions/use-params)は、Client ComponentsでURLパスに含まれるDynamic Params（e.g. `/posts/[slug]`の`[slug]`部分）を参照するためのhooksです。

```tsx
"use client";

import { useParams } from "next/navigation";

export default function ExampleClientComponent() {
  const params = useParams<{ tag: string; item: string }>();

  // Route: /shop/[tag]/[item]
  // URL  : /shop/shoes/nike-air-max-97
  console.log(params); // { tag: 'shoes', item: 'nike-air-max-97' }

  // ...
}
```

#### `searchParams` props

[`searchParams` props↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional)は、URLのクエリー文字列を参照するためのpropsです。`searchParams` propsでは、クエリー文字列のkey-value相当なオブジェクトが提供されます。

| URL                             | `searchParams` props             |
| ------------------------------- | -------------------------------- |
| `/products?id=1`                | `{ id: "1" }`                    |
| `/products?id=1&sort=recommend` | `{ id: "1", sort: "recommend" }` |
| `/products?id=1&id=2`           | `{ id: ["1", "2"] }`             |

```tsx
type SearchParamsValue = string | string[] | undefined;

export default async function Page({
  searchParams,
}: {
  searchParams: Promise<{
    sort?: SearchParamsValue;
    id?: SearchParamsValue;
  }>;
}) {
  const { sort, id } = await searchParams;
  // ...
}
```

#### `useSearchParams()`

[`useSearchParams()`↗︎](https://nextjs.org/docs/app/api-reference/functions/use-search-params)は、Client ComponentsでURLのクエリー文字列を参照するためのhooksです。

```tsx
"use client";

import { useSearchParams } from "next/navigation";

export default function SearchBar() {
  const searchParams = useSearchParams();
  const search = searchParams.get("search");

  // URL -> `/dashboard?search=my-project`
  console.log(search); // 'my-project'

  // ...
}
```

### ヘッダー情報の参照

#### `headers()`

[`headers()`↗︎](https://nextjs.org/docs/app/api-reference/functions/headers)は、リクエストヘッダーを参照するための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

```tsx
import { headers } from "next/headers";

export default async function Page() {
  const headersList = await headers();
  const referrer = headersList.get("referrer");

  return <div>Referrer: {referrer}</div>;
}
```

### クッキー情報の参照と変更

#### `cookies()`

[`cookies()`↗︎](https://nextjs.org/docs/app/api-reference/functions/cookies)は、Cookie情報の参照や変更を担うオブジェクトを取得するための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

:::message
Cookieの`.set()`や`.delete()`といった操作は、Server ActionsやRoute Handlerでのみ利用でき、Server Componentsでは利用できません。詳しくは[Server Componentsの純粋性](part_4_pure_server_components)を参照ください。
:::

```tsx :app/page.tsx
import { cookies } from "next/headers";

export default async function Page() {
  const cookieStore = await cookies();
  const theme = cookieStore.get("theme");
  return "...";
}
```

```ts :app/actions.ts
"use server";

import { cookies } from "next/headers";

async function create(data) {
  const cookieStore = await cookies();
  cookieStore.set("name", "lee");

  // ...
}
```

### レスポンスのステータスコード

Next.jsはStreamingをサポートしているため、確実にHTTPステータスコードを設定する手段がありません。その代わりに、`notFound()`や`redirect()`といった関数でブラウザに対してエラーやリダイレクトを示すことができます。

これらを呼び出した際には、まだHTTPレスポンスの送信がまだ開始されていなければステータスコードが設定され、すでにクライアントにステータスコードが送信されていた場合には`<meta>`タグが挿入されてブラウザにこれらの情報が伝えられます。

#### `notFound()`

[`notFound()`↗︎](https://nextjs.org/docs/app/api-reference/functions/not-found)は、ページが存在しないことをブラウザに示すための関数です。Server Componentsで利用することができます。この関数が呼ばれた際には、該当Routeの`not-found.tsx`が表示されます。

```tsx
import { notFound } from "next/navigation";

// ...

export default async function Profile({ params }: { params: { id: string } }) {
  const user = await fetchUser(params.id);

  if (!user) {
    notFound();
  }

  // ...
}
```

#### `redirect()`

[`redirect()`↗︎](https://nextjs.org/docs/app/api-reference/functions/redirect)は、リダイレクトを行うための関数です。この関数はServer ComponentsやClient Components、Server FunctionsやRouter Handlerなど、多くの場所で利用できます。

```tsx
import { redirect } from "next/navigation";

// ...

export default async function Profile({ params }: { params: { id: string } }) {
  const team = await fetchTeam(params.id);
  if (!team) {
    redirect("/login");
  }

  // ...
}
```

#### `permanentRedirect()`

[`permanentRedirect()`↗︎](https://nextjs.org/docs/app/api-reference/functions/permanentRedirect)は、永続的なリダイレクトを行うための関数です。この関数はServer ComponentsやClient Components、Server FunctionsやRouter Handlerなど、多くの場所で利用できます。

```tsx
import { permanentRedirect } from "next/navigation";

// ...

export default async function Profile({ params }: { params: { id: string } }) {
  const team = await fetchTeam(params.id);
  if (!team) {
    permanentRedirect("/login");
  }

  // ...
}
```

### `unauthorized()`

[`unauthorized()`↗︎](https://nextjs.org/docs/app/api-reference/functions/unauthorized)は認証エラーを示すための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

:::message
このAPIは執筆時現在、実験的機能です。
:::

```tsx
import { verifySession } from "@/app/lib/dal";
import { unauthorized } from "next/navigation";

export default async function DashboardPage() {
  const session = await verifySession();
  if (!session) {
    unauthorized();
  }

  // ...
}
```

### `forbidden()`

[`forbidden()`↗︎](https://nextjs.org/docs/app/api-reference/functions/forbidden)は認可エラーを示すための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

:::message
このAPIは執筆時現在、実験的機能です。
:::

```tsx
import { verifySession } from "@/app/lib/dal";
import { forbidden } from "next/navigation";

export default async function AdminPage() {
  const session = await verifySession();
  if (session.role !== "admin") {
    forbidden();
  }

  // ...
}
```

### その他

筆者が主要なAPIとして認識してるものは上記に列挙しましたが、Next.jsでは他にも必要に応じて様々なAPIが提供されています。上記にないユースケースで困った場合には、公式ドキュメントより検索してみましょう。

https://nextjs.org/docs

## トレードオフ

### `req`拡張によるセッション情報の持ち運び

従来`req`オブジェクトは、3rd partyライブラリが拡張して`req.session`にセッション情報を格納するような実装がよく見られました。Next.jsではこのような実装はできず、これに代わるセッション管理の仕組みなどを実装する必要があります。

以下は、GitHub OAuthアプリとして実装したサンプル実装の一部です。`sessionStore.get()`でRedisに格納したセッション情報を取得できます。

https://github.com/AkifumiSato/nextjs-book-oauth-app-example/blob/main/app/api/github/callback/route.ts#L12

セッション管理の実装が必要な方は、必要に応じて上記のリポジトリを参考にしてみてください。
