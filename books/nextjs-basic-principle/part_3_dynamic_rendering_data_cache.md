---
title: "Dynamic RenderingとData Cache"
---

## 要約

Dynamic Renderingなページでは、データフェッチ単位のキャッシュであるData Cacheを活用してパフォーマンスを最適化しましょう。

## 背景

[Static RenderingとFull Route Cache](part_3_static_rendering_full_route_cache)で述べた通り、Next.jsでは可能な限りStatic Renderingにすることが推奨されています。しかし、アプリケーションによってはユーザー情報を含むページなど、Dynamic Renderingが必要な場合も当然考えられます。

Dynamic Renderingはリクエストごとにレンダリングされるので、できるだけ早く完了する必要があります。この際最もボトルネックになりやすいのが**データフェッチ処理**です。

:::message
RouteをDynamic Renderingに切り替える方法は前の章の[Static RenderingとFull Route Cache](part_3_static_rendering_full_route_cache#背景)で解説していますので、そちらをご参照ください。
:::

## 設計・プラクティス

[Data Cache↗︎](https://nextjs.org/docs/app/guides/caching#data-cache)はデータフェッチ処理の結果をキャッシュするもので、サーバー側に永続化され**リクエストやユーザーを超えて共有**されます。

Dynamic RenderingではNext.jsサーバーへのリクエストごとにレンダリングを行いますが、その際必ずしも全てのデータフェッチを実行しなければならないとは限りません。ユーザー情報に紐づくようなデータフェッチとそうでないものを切り分けて、後者に対しData Cacheを活用することで、Dynamic Renderingの高速化やAPIサーバーの負荷軽減などが見込めます。

Data Cacheができるだけキャッシュヒットするよう、データフェッチごとに適切な設定を心がけましょう。

### Next.jsと`fetch()`

サーバー上で実行される`fetch()`は[Next.jsによって拡張↗︎](https://nextjs.org/docs/app/api-reference/functions/fetch#fetchurl-options)されており、Data Cacheに関するオプションが組み込まれています。デフォルトではキャッシュは無効ですが、第2引数のオプション指定によってキャッシュ挙動を変更することが可能です。

```ts
fetch(`https://...`, {
  cache: "force-cache", // or "no-store",
});
```

:::message alert
v14以前において、[`cache`オプション↗︎](https://nextjs.org/docs/app/api-reference/functions/fetch#optionscache)のデフォルトは`"force-cache"`でした。v15ではデフォルトでキャッシュが無効になるよう変更されていますが、デフォルトではStatic Renderingとなっています。Dynamic Renderingに切り替えるには明示的に`"no-store"`を指定する必要があるので、注意しましょう。
:::

```ts
fetch(`https://...`, {
  next: {
    revalidate: false, // or number,
  },
});
```

`next.revalidate`は文字通りrevalidateされるまでの時間を設定できます。

```ts
fetch(`https://...`, {
  next: {
    tags: [tagName], // string[]
  },
});
```

`next.tags`には配列で**tag**を複数指定することができます。これは後述の`revalidateTag()`によって指定したtagに関連するData Cacheをrevalidateする際に利用されます。

### `unstable_cache()`

`unstable_cache()`を使うことで、DBアクセスなどでもData Cacheを利用することが可能です。

:::message alert
`unstable_cache()`は将来的に[`"use cache"`↗︎](https://nextjs.org/docs/app/api-reference/directives/use-cache)に置き換えられる予定です。
:::

```tsx
import { getUser } from "./fetcher";
import { unstable_cache } from "next/cache";

const getCachedUser = unstable_cache(
  getUser, // DBアクセス
  ["my-app-user"], // key array
  {
    tags: ["users"], // cache revalidate tags
    revalidate: 60, // revalidate time(s)
  },
);

export default async function Component({ userID }) {
  const user = await getCachedUser(userID);
  // ...
}
```

### オンデマンドrevalidate

[Static RenderingとFull Route Cache](part_3_static_rendering_full_route_cache)でも述べた通り、[`revalidatePath()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)や[`revalidateTag()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)をServer Actions^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]や[Route Handlers↗︎](https://nextjs.org/docs/app/getting-started/route-handlers-and-middleware)で呼び出すことで、関連するData CacheやFull Route Cacheをrevalidateすることができます。

```ts
"use server";

import { revalidatePath } from "next/cache";

export async function action() {
  // ...

  revalidatePath("/products");
}
```

これらは特に何かしらのデータ操作が発生した際に利用されることを想定したrevalidateです。サイト内ではServer Actionsを、外部で発生したデータ操作に対してはRoute Handlersからrevalidateすることが推奨されます。

Next.jsでのデータ操作に関する詳細は、後述の[データ操作とServer Actions](part_3_data_mutation)にて解説します。

::::details 余談: Data Cacheと`revalidatePath()`の仕組み

Data Cacheにはデフォルトのtagとして、Route情報を元にしたタグが内部的に設定されており、`revalidatePath()`はこの特殊なタグを元に関連するData Cacheのrevalidateを実現しています。

より詳細にrevalidateの仕組みを知りたい方は、過去に筆者が調査した際の以下の記事をぜひご参照ください。

https://zenn.dev/akfm/articles/nextjs-revalidate

::::

## トレードオフ

### Data CacheのオプトアウトとDynamic Rendering

`fetch()`のオプションで`cahce: "no-store"`か`next.revalidate: 0`を設定することでData Cacheをオプトアウトすることができますが、これは同時にRouteが**Dynamic Renderingに切り替わる**ことにもなります。

これらを設定する時は本当にDynamic Renderingにしなければいけないのか、よく考えて設定しましょう。
