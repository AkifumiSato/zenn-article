---
title: "dynamic renderingとData Cache"
---

## 要約

Data Cacheを活用して、dynamic rendering時のパフォーマンスを最適化しましょう。

## 背景

[static renderingとFull Route Cache](part_2_static_rendering_full_route_cache)で述べた通り、App Routerでは可能な限りstatic renderingにすることが推奨されています。しかし、アプリケーションによってはユーザー情報を含むページなどdynamic renderingが必要な場合もあります。

dynamic renderingはリクエストごとにレンダリングされるのでできるだけ早く完了する必要があります。この際最もパフォーマンスボトルネックになりやすいのが**データフェッチ処理**です。

:::message
Routeをdynamic renderingに切り替える方法は前の章の[static renderingとFull Route Cache](part_2_static_rendering_full_route_cache#背景)で解説していますので、そちらをご参照ください。
:::

## 設計・プラクティス

[Data Cache](https://nextjs.org/docs/app/building-your-application/caching#data-cache)はデータフェッチ処理の結果をキャッシュするもので、サーバー側に永続化されリクエストやユーザーを超えて共有されます。これを活用することでdynamic renderingの高速化やAPI負荷軽減などが見込めます。

Data Cacheができるだけキャッシュヒットするよう適切な設定を心がけましょう。

### Next.jsサーバー上の`fetch()`

サーバー上で実行される`fetch()`は[Next.jsによって拡張](https://nextjs.org/docs/app/api-reference/functions/fetch#fetchurl-options)されておりData Cacheが組み込まれています。デフォルトではキャッシュは永続化されますが、第2引数のオプション指定によってキャッシュ挙動を変更することが可能です。

```ts
fetch(`https://...`, {
  cache: "force-cache", // or "no-store",
});
```

`cache`に`force-cache`か`no-store`を指定でき、これによりキャッシュを有効・無効にすることができます。

:::message
Next.jsの`v15.0.0-rc.0`では`fetch`の[デフォルトが`no-store`](https://nextjs.org/blog/next-15-rc#caching-updates)に変更されました。
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

`next.tags`には配列でタグを複数指定することができます。これは後述の`revalidateTag()`によって指定したタグに関連するData Cacheをrevalidateする際に利用されます。

### `unstable_cache()`

[`unstable_cache()`](https://nextjs.org/docs/app/api-reference/functions/unstable_cache)を使うことで、DBアクセスやGraphQLでもData Cacheを利用することが可能です。

```tsx
import { getUser } from "./fetcher";
import { unstable_cache } from "next/cache";

const getCachedUser = unstable_cache(
  getUser, // DBアクセス
  ["my-app-user"], // key array
);

export default async function Component({ userID }) {
  const user = await getCachedUser(userID);
  // ...
}
```

:::message alert
`unstable_cache()`はAPI名の通り安定版ではなく、今後変更される可能性があります。
:::

### オンデマンドrevalidate

[static renderingとFull Route Cache](part_2_static_rendering_full_route_cache)でも述べた通り、[`revalidatePath()`](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)や[`revalidateTag()`](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)を[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)や[Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)で呼び出すことで、関連するData CacheやFull Route Cacheをrevalidateすることができます。

```ts
"use server";

import { revalidatePath } from "next/cache";

export async function action() {
  // ...

  revalidatePath("/products");
}
```

これらは特に何かしらのデータ操作が発生した際に利用されることを想定したrevalidateです。App Routerでのデータ操作に関する詳細は[データ操作とServer Actions](part_2_data_mutation_inner)と[外部で発生したデータ操作](part_2_data_mutation_outer)にて解説します。

#### Data Cacheと`revalidatePath()`

Data CacheにはデフォルトのタグとしてRoute情報を元にしたタグがNext.js内部より設定されており、`revalidatePath()`はこの特殊なタグを元に関連するData Cacheのrevalidateを実現しています。

:::message
より詳細にrevalidateの仕組みを知りたい方は、過去に筆者が調査した際の[こちらの記事](https://zenn.dev/akfm/articles/nextjs-revalidate)をぜひご参照ください。
:::

## トレードオフ

### Data Cacheのオプトアウトとdynamic rendering

`fetch()`のオプションで`cahce: "no-store"`か`next.revalidate: 0`を設定することでData Cacheをオプトアウトすることができますが、これは同時にRouteが**dynamic renderingに切り替わる**ことにもなります。

これらを設定する時は本当にdynamic renderingにしなければいけないのか、よく考えて設定しましょう。
