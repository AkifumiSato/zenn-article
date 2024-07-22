---
title: "static renderingとFull Route Cache"
---

## 要約

revalidateを駆使して可能な限りstatic renderingにし、Full Route Cacheを活用しましょう。

## 背景

従来Pages Routerではサーバー側のレンダリングについて、[SSR](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)という3つのレンダリングモデルをサポートしてきました。App Routerでは上記同等のレンダリングをサポートしつつ、オンデマンドなrevalidateがより整理されてSSGとISRを大きく区別する意味がなくなったため、これらをまとめて**static rendering**・従来のSSR相当を**dynamic rendering**と呼称する形で再定義されました。

| レンダリング                                                                                                                   | タイミング            | Pages Routerとの比較 |
| ------------------------------------------------------------------------------------------------------------------------------ | --------------------- | -------------------- |
| [static rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default) | build時やrevalidate後 | SSG・ISR相当         |
| [dynamic rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)       | ユーザーリクエスト時  | SSR相当              |

:::message
Server Componentsは[Soft Navigation](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#5-soft-navigation)時も実行されるので必ずしもSSR・SSG・ISRと比較できるものではないですが、ここでは簡略化して比較しています。
:::

App Routerは**デフォルトでstatic rendering**となっており、**dynamic renderingはオプトイン**になっています。dynamic renderingにオプトインする方法は以下の通りです。

### dynamic functions

`cookies()`/`headers()`などの[dynamic functions](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-functions)と呼ばれる関数を呼び出すと、dynamic renderingとなります。

```ts
// page.tsx
import { cookies } from "next/headers";

export default function Page() {
  const cookieStore = cookies();
  const sessionId = cookieStore.get("session-id");

  return "...";
}
```

### `fetch()` Data Cacheオプトアウト

`fetch()`のオプションで[Data Cache](https://nextjs.org/docs/app/building-your-application/caching#data-cache)をオプトアウトした場合、dynamic renderingとなります。キャッシュをオプトアウトするには
`cache: "no-store"`か`next: { revalidate: 0 }`を指定する必要があります。

```ts
// page.tsx
export default async function Page({
  params: { id },
}: {
  params: { id: string };
}) {
  const res = await fetch(`https://dummyjson.com/products/${id}`, {
    cache: "no-store",
  });
  const product = await res.json();

  return "...";
}
```

:::message
Next.jsの`v15.0.0-rc.0`では`fetch`のデフォルトが`cache: "no-store"`に変更されていますが、明示的に指定しないとdynamic renderingにならないという仕様になっています。

デフォルトの仕様については[RC中に変更される可能性](https://x.com/feedthejim/status/1794778189354705190)が示唆されていますが、本書執筆時点では変更されるかは不明です。
:::

### Route Segment Config

[Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)を利用してdynamic renderingに切り替えることもできます。具体的には、`page.tsx`や`layout.tsx`に以下どちらかを設定することでdynamic renderingを強制できます。

```tsx
// layout.tsx | page.tsx
export const dynamic = "force-dynamic";
```

```tsx
// layout.tsx | page.tsx
export const revalidate = 0; // 1以上でstatic rendering
```

:::message
`layout.tsx`に設定したRoute Segment ConfigはLayoutが利用される下層ページにも適用されるため、注意しましょう。
:::

### `unstable_noStore()`

末端のコンポーネントでdynamic renderingを強制したいがdynamic functionsや`no-store`な`fetch()`を使っていない場合には、`unstable_noStore()`を呼び出すことでdynamic renderingに切り替えることができます。

```ts
import { unstable_noStore as noStore } from "next/data-store";

export function LeafComponent() {
  noStore();

  // DBアクセスなど

  return "...";
}
```

## 設計・プラクティス

static renderingは耐障害性・パフォーマンスに優れています。ユーザーリクエスト毎にレンダリングが必要なら前述の方法でdynamic renderingにオプトインする必要がありますが、それ以外のケースについてApp Routerでは**可能な限りstatic renderingにする**ことが推奨されています。

static renderingのレンダリング結果のキャッシュは[Full Route Cache](https://nextjs.org/docs/app/building-your-application/caching#full-route-cache)と呼ばれています。App Routerではstatic renderingを活用するために、Full Route Cacheのオンデマンドrevalidateや時間ベースでのrevalidateといったよくあるユースケースをフォローし、従来のSSGのように変更があるたびにデプロイが必要といったことがないように設計されています。

### オンデマンドrevalidate

[`revalidatePath()`](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)や[`revalidateTag()`](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)を[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)や[Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)で呼び出すことで、関連するData CacheやFull Route Cacheをrevalidateすることができます。

```ts
"use server";

import { revalidatePath } from "next/cache";

export async function action() {
  // ...

  revalidatePath("/products");
}
```

これらは特に何かしらのデータ操作が発生した際に利用されることを想定したrevalidateです。App Routerでのデータ操作に関する詳細は[データ操作とServer Actions](part_3_data_mutation_inner)と[外部で発生したデータ操作](part_3_data_mutation_outer)にて解説します。

### 時間ベースrevalidate

Route Segment Configの[revalidate](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#revalidate)を指定することでFull Route Cacheや関連するData Cacheを時間ベースでrevalidateすることができます。

```tsx
// layout.tsx | page.tsx
export const revalidate = 10; // 10s
```

:::message
重複になりますが、`layout.tsx`に`revalidate`を設定するとLayoutが利用される下層ページにも適用されるため、注意しましょう。
:::

非常に短い時間、例えば1秒設定するだけで秒間数百のリクエストが発生してもレンダリングを1つにまとめることができるので、バックエンドAPIへの負荷軽減・安定したパフォーマンスを実現できます。

## トレードオフ

### 予期せぬdynamic renderingとパフォーマンス劣化

Route Segment Configや`unstable_noStore()`によってdynamic renderingを利用する場合、開発者は明らかにdynamic renderingを意識して使うのでこれらが及ぼす影響を見誤ることは少ないと考えられます。一方、dynamic functionsは「cookieを利用したい」、`cache: "no-store"`な`fetch`は「Data Cacheを使いたくない」などの主目的が別にあり、これに伴って副次的にdynamic renderingに切り替わるため、開発者は影響範囲に注意する必要があります。

特に、Data Cacheなどを適切に設定できていないとdynamic renderingに切り替わった際にページ全体のパフォーマンス劣化につながる可能性があります。こちらについての詳細は後述の[dynamic renderingとData Cache](part_3_dynamic_rendering_data_cache)をご参照ください。

### static/dynamic rendering境界とPPR

Next.jsのv14時点ではdynamic renderingはRoute単位(`page.tsx`や`layout.tsx`)でしか切り替えられませんが、experimentalフラグで**PPR**(Partial Pre-Rendering)を有効にするにより、文字通りPartial(部分的)にdynamic renderingへの切り替えが可能になります。PPRではstatic/dynamic renderingの境界を`<Suspense>`によって定義します。

PPRについては後述の[PPRの章](part_2_partial_pre_rendering)や筆者の過去記事である以下をご参照ください。

https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering
