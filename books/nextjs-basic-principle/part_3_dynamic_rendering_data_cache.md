---
title: "Dynamic RenderingとData Cache"
---

## 要約

Dynamic Renderingなページでは、データフェッチ単位のキャッシュであるData Cacheを活用してパフォーマンスを最適化しましょう。

## 背景

[_Static RenderingとFull Route Cache_](part_3_static_rendering_full_route_cache)で述べた通り、App Routerでは可能な限りStatic Renderingにすることが推奨されています。しかし、アプリケーションによってはユーザー情報を含むページなど、Dynamic Renderingが必要な場合も当然考えられます。

Dynamic Renderingはリクエストごとにレンダリングされるので、できるだけ早く完了する必要があります。この際最もボトルネックになりやすいのが**データフェッチ処理**です。

:::message
RouteをDynamic Renderingに切り替える方法は前の章の[_Static RenderingとFull Route Cache_](part_3_static_rendering_full_route_cache#背景)で解説していますので、そちらをご参照ください。
:::

## 設計・プラクティス

[Data Cache](https://nextjs.org/docs/app/building-your-application/caching#data-cache)はデータフェッチ処理の結果をキャッシュするもので、サーバー側に永続化され**リクエストやユーザーを超えて共有**されます。

Dynamic RenderingはNext.jsサーバーへのリクエストごとにレンダリングを行いますが、その際必ずしも全てのデータフェッチを実行しなければならないとは限りません。ユーザー情報に紐づくようなデータフェッチとそうでないものを切り分けて、後者に対しData Cacheを活用することで、Dynamic Renderingの高速化やAPIサーバーの負荷軽減などが見込めます。

Data Cacheができるだけキャッシュヒットするよう、データフェッチごとに適切な設定を心がけましょう。

### Next.jsサーバー上の`fetch()`

サーバー上で実行される`fetch()`は[Next.jsによって拡張](https://nextjs.org/docs/app/api-reference/functions/fetch#fetchurl-options)されており、Data Cacheに関するオプションが組み込まれています。デフォルトではキャッシュは無効ですが、第2引数のオプション指定によってキャッシュ挙動を変更することが可能です。

```ts
fetch(`https://...`, {
  cache: "force-cache", // or "no-store",
});
```

:::message alert
v14以前において、[`cache`オプション](https://nextjs.org/docs/app/api-reference/functions/fetch#optionscache)のデフォルトは`"force-cache"`でした。v15ではデフォルトでキャッシュが無効になるよう変更されていますが、デフォルトではStatic Renderingとなっています。Dynamic Renderingに切り替えるには明示的に`"no-store"`を指定する必要があるので、注意しましょう。
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

:::message alert
`unstable_cache()`はv15で非推奨となりました。ただし、移行先である`"use cache"`は[Dynamic IO](https://nextjs.org/docs/app/api-reference/config/next-config-js/dynamicIO)でのみ有効で、Dynamic IO自体がまだ実験的機能の段階です。
今後Dynamic IOが安定化したら移行を考えましょう。
:::

### オンデマンドrevalidate

[_Static RenderingとFull Route Cache_](part_3_static_rendering_full_route_cache)でも述べた通り、`revalidatePath()`や`revalidateTag()`を[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)や[Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)で呼び出すことで、関連するData CacheやFull Route Cacheをrevalidateすることができます。

```ts
"use server";

import { revalidatePath } from "next/cache";

export async function action() {
  // ...

  revalidatePath("/products");
}
```

これらは特に何かしらのデータ操作が発生した際に利用されることを想定したrevalidateです。サイト内からのデータ操作にはServer Actionsを、外部で発生したデータ操作に対してはRoute Handlersからrevalidateすることが推奨されます。

App Routerでのデータ操作に関する詳細は、後述の[_データ操作とServer Actions_](part_3_data_mutation)にて解説します。

#### Data Cacheと`revalidatePath()`

余談ですが、Data Cacheにはデフォルトのtagとして、Route情報を元にしたタグが内部的に設定されており、`revalidatePath()`はこの特殊なタグを元に関連するData Cacheのrevalidateを実現しています。

より詳細にrevalidateの仕組みを知りたい方は、過去に筆者が調査した際の以下の記事をぜひご参照ください。

https://zenn.dev/akfm/articles/nextjs-revalidate

## トレードオフ

### Data CacheのオプトアウトとDynamic Rendering

`fetch()`のオプションで`cahce: "no-store"`か`next.revalidate: 0`を設定することでData Cacheをオプトアウトすることができますが、これは同時にRouteが**Dynamic Renderingに切り替わる**ことにもなります。

これらを設定する時は本当にDynamic Renderingにしなければいけないのか、よく考えて設定しましょう。

また、Next.jsではv14以降、Static RenderingとDynamic Renderingを1つのRouteで混在させることができる**Partial Pre Rendering**(PPR)をexperimentalで提供しています。PPRでは、Suspense境界単位でDynamic Renderingにオプトインすることができます。PPRのより詳細な内容については、後述の[_Partial Pre Rendering(PPR)_](part_4_partial_pre_rendering)で解説します。
