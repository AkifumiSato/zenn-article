---
title: "リクエスト情報の参照とレスポンス"
---

## 要約

App Routerでは他フレームワークにあるようなリクエストオブジェクト(`req`)やレスポンスオブジェクト(`res`)を参照することはできません。代わりに必要な情報を参照するためのhooksや関数などのAPIが提供されています。

- [`searchParams` props](https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional)
- [`cookies()`](https://nextjs.org/docs/app/api-reference/functions/cookies)
- [`headers()`](https://nextjs.org/docs/app/api-reference/functions/headers)
- [`useParams()`](https://nextjs.org/docs/app/api-reference/functions/use-params)
- [`useSearchParams()`](https://nextjs.org/docs/app/api-reference/functions/use-search-params)
- [`notFound()`](https://nextjs.org/docs/app/api-reference/functions/not-found)
- [`redirect()`](https://nextjs.org/docs/app/api-reference/functions/redirect)
- [`permanentRedirect()`](https://nextjs.org/docs/app/api-reference/functions/permanentRedirect)
- [その他](https://nextjs.org/docs/app/api-reference/functions)

## 背景

Pages Routerはじめ従来のフレームーワークでは、リクエストオブジェクト(`req`)やレスポンスオブジェクト(`res`)を参照することで様々な情報にアクセスしたり、レスポンスをカスタマイズすることができました。

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

App Routerでは上記のようなリクエスト単位のオブジェクトを提供する代わりに、それぞれ必要な情報を参照するための関数などのAPIが提供されています。

:::message
Server Componentsでリクエスト時の情報を参照する関数などは[Dynamic Functions](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-functions)と呼ばれ、これらを利用するとRoute全体が[Dynamic Rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)となります。
:::

### `params` props

Dynamic RoutesのURLパスの情報は[`params` props](https://nextjs.org/docs/app/api-reference/file-conventions/page#params-optional)で提供されます。以下は`/posts/[slug]`と`/posts/[slug]/comments/[commentId]`というルーティングがあった場合の`params`の例です。

| URL                        | `params` props                       |
| -------------------------- | ------------------------------------ |
| `/posts/hoge`              | `{ slug: "hoge" }`                   |
| `/posts/hoge/comments/111` | `{ slug: "hoge", commentId: "111" }` |

### `searchParams` props

[`searchParams` props](https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional)は、URLのGETパラメータを参照するためのpropsです。`searchParams` propsでは、GETパラメータのkey-value相当なオブジェクトが提供されます。

| URL                             | `searchParams` props             |
| ------------------------------- | -------------------------------- |
| `/products?id=1`                | `{ id: "1" }`                    |
| `/products?id=1&sort=recommend` | `{ id: "1", sort: "recommend" }` |
| `/products?id=1&id=2`           | `{ id: ["1", "2"] }`             |

```tsx
export default function Page({
  params,
  searchParams,
}: {
  params: { slug: string };
  searchParams: {
    [key: string]: string | string[] | undefined;
  };
}) {
  // ...
}
```

### `cookies()`

[`cookies()`](https://nextjs.org/docs/app/api-reference/functions/cookies)は、Cookie情報を参照するための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

:::message
`cookies().set()`をはじめとしたCookieの操作は、Server ActionsやRoute Handlerでのみ利用でき、**Server Componentsでは利用できません**。
:::

```tsx
import { cookies } from "next/headers";

export default function Page() {
  const cookieStore = cookies();
  const theme = cookieStore.get("theme");
  return "...";
}
```

### `headers()`

[`headers()`](https://nextjs.org/docs/app/api-reference/functions/headers)は、リクエストヘッダーを参照するための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

```tsx
import { headers } from "next/headers";

export default function Page() {
  const headersList = headers();
  const referer = headersList.get("referer");

  return <div>Referer: {referer}</div>;
}
```

### `useParams()`

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

### `useSearchParams()`

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

### `notFound()`

[`notFound()`](https://nextjs.org/docs/app/api-reference/functions/not-found)は、ページが存在しないことをブラウザに示すための関数です。Server Componentsで利用することができます。

:::message
多くの場合、`notFound()`はHTTP Status Codeとして404 Not Foundを返しますが、`<Suspens>`内などで利用すると200 OKを返すことがあります。この際、`<meta name="robots" content="noindex" />`タグを挿入してGoogleクローラなどに対してIndexingの必要がないことを示します。
:::

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

### `redirect()`

[`redirect()`](https://nextjs.org/docs/app/api-reference/functions/redirect)は、リダイレクトを行うための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

:::message
この関数は必ずしもHTTP Status Codeではなく、`<meta>`タグでリダイレクトをブラウザに指示することがあります。
:::

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

### `permanentRedirect()`

[`permanentRedirect()`](https://nextjs.org/docs/app/api-reference/functions/permanentRedirect)は、永続的なリダイレクトを行うための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

:::message
この関数は必ずしもHTTP Status Codeではなく、`<meta>`タグでリダイレクトをブラウザに指示することがあります。
:::

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

特になし
