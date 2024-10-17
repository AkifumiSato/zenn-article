---
title: "リクエストの参照とレスポンスの操作"
---

## 要約

App Routerでは他フレームワークにあるようなリクエストオブジェクト(`req`)やレスポンスオブジェクト(`res`)を参照することはできません。代わりに必要な情報を参照するためのAPIが提供されています。

## 背景

Pages Router然り従来のNode.jsのフレームーワークでは、リクエストオブジェクト(`req`)やレスポンスオブジェクト(`res`)を参照することで様々な情報にアクセスしたり、レスポンスをカスタマイズするような設計が広く使われてきました。

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

しかし、App Routerではこれらのオブジェクトを参照することはできません。

## 設計・プラクティス

App Routerではリクエストやレスポンスオブジェクトを提供する代わりに、必要な情報を参照するためのAPIが提供されています。

:::message
Server Componentsでリクエスト時の情報を参照する関数は[Dynamic Functions](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-functions)と呼ばれ、これらを利用するとRoute全体が[Dynamic Rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)となります。
:::

### URL情報の参照

#### `params` props

Dynamic RoutesのURLパスの情報は[`params` props](https://nextjs.org/docs/app/api-reference/file-conventions/page#params-optional)で提供されます。以下は`/posts/[slug]`と`/posts/[slug]/comments/[commentId]`というルーティングがあった場合の`params`の例です。

| URL                        | `params` props                       |
| -------------------------- | ------------------------------------ |
| `/posts/hoge`              | `{ slug: "hoge" }`                   |
| `/posts/hoge/comments/111` | `{ slug: "hoge", commentId: "111" }` |

```tsx
export default function Page({
  params,
}: {
  params: {
    slug: string;
    commentId: string;
  };
}) {
  // ...
}
```

#### `searchParams` props

[`searchParams` props](https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional)は、URLのGETパラメータを参照するためのpropsです。`searchParams` propsでは、GETパラメータのkey-value相当なオブジェクトが提供されます。

| URL                             | `searchParams` props             |
| ------------------------------- | -------------------------------- |
| `/products?id=1`                | `{ id: "1" }`                    |
| `/products?id=1&sort=recommend` | `{ id: "1", sort: "recommend" }` |
| `/products?id=1&id=2`           | `{ id: ["1", "2"] }`             |

```tsx
type SearchParamsValue = string | string[] | undefined;

export default function Page({
  searchParams,
}: {
  searchParams: {
    sort?: SearchParamsValue;
    id?: SearchParamsValue;
  };
}) {
  // ...
}
```

#### `useParams()`

[`useParams()`](https://nextjs.org/docs/app/api-reference/functions/use-params)は、Client ComponentsでURLパスに含まれるDynamic Params（e.g. `/posts/[slug]`の`[slug]`部分）を参照するためのhooksです。

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

#### `useSearchParams()`

[`useSearchParams()`](https://nextjs.org/docs/app/api-reference/functions/use-search-params)は、Client ComponentsでURLのGETパラメータを参照するためのhooksです。

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

[`headers()`](https://nextjs.org/docs/app/api-reference/functions/headers)は、リクエストヘッダーを参照するための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

```tsx
import { headers } from "next/headers";

export default function Page() {
  const headersList = headers();
  const referer = headersList.get("referer");

  return <div>Referer: {referer}</div>;
}
```

### クッキー情報の参照と変更

#### `cookies()`

[`cookies()`](https://nextjs.org/docs/app/api-reference/functions/cookies)は、Cookie情報の参照や変更を担うオブジェクトを取得するための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

:::message
`cookies().set()`や`cookies().delete()`といったCookieの操作は、Server ActionsやRoute Handlerでのみ利用でき、Server Componentsでは利用できません。詳しくは[_Server Componentsの純粋性_](part_4_pure_server_components)を参照ください。
:::

```tsx :app/page.tsx
import { cookies } from "next/headers";

export default function Page() {
  const cookieStore = cookies();
  const theme = cookieStore.get("theme");
  return "...";
}
```

```ts :app/actions.ts
"use server";

import { cookies } from "next/headers";

async function create(data) {
  cookies().set("name", "lee");

  // ...
}
```

### レスポンスのStatus Code

App RouterはStreamingをサポートしているため、確実にHTTP Status Codeを設定する手段がありません。その代わりに、`notFound()`や`redirect()`といった関数でブラウザに対してリダイレクトやエラーを示すことができます。

これらを呼び出した際には、まだHTTP Status Codeがクライアントに返されてなければ適切設定し、すでにクライアントにStatus Codeが送信されていた場合には`<meta>`タグを挿入してブラウザにこれらの情報を伝えます。

#### `notFound()`

[`notFound()`](https://nextjs.org/docs/app/api-reference/functions/not-found)は、ページが存在しないことをブラウザに示すための関数です。Server Componentsで利用することができます。

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

[`redirect()`](https://nextjs.org/docs/app/api-reference/functions/redirect)は、リダイレクトを行うための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

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

[`permanentRedirect()`](https://nextjs.org/docs/app/api-reference/functions/permanentRedirect)は、永続的なリダイレクトを行うための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

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

### その他

筆者が主要なAPIとして認識してるものは上記に列挙しましたが、App Routerでは他にも必要に応じて様々なAPIが提供されています。上記にないユースケースで困った場合には、公式ドキュメントより検索してみましょう。

https://nextjs.org/docs

## トレードオフ

### `req`拡張によるセッション情報の持ち運び

従来`req`オブジェクトは、3rd partyライブラリが拡張して`req.session`にセッション情報を格納するような実装がよく見られました。App Routerではこのような実装はできず、これに代わるセッション管理の仕組みなどを実装する必要があります。

以下は、GitHub OAuthアプリとして実装したサンプル実装の一部です。`sessionStore.get()`でRedisに格納したセッション情報を取得できます。

https://github.com/AkifumiSato/nextjs-book-oauth-app-example/blob/main/app/api/github/callback/route.ts#L12

セッション管理の実装が必要な方は、必要に応じて上記のリポジトリを参考にしてみてください。
