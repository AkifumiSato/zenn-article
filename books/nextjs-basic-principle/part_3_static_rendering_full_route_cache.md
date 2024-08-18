---
title: "Static RenderingとFull Route Cache"
---

## 要約

Static Renderingでは、htmlやRSC PayloadのキャッシュであるFull Route Cacheを生成します。Full Route Cacheは短い期間でrevalidateも可能なので、ユーザー固有の情報を含まないようなページは積極的にFull Route Cacheを活用しましょう。

## 背景

従来Pages Routerではサーバー側のレンダリングについて、[SSR](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)という3つのレンダリングモデル^[[こちらの記事](https://vercel.com/blog/partial-prerendering-with-next-js-creating-a-new-default-rendering-model)より引用。レンダリング戦略とも呼ばれます。]をサポートしてきました。App Routerでは上記相当のレンダリングをサポートしつつ、オンデマンドなrevalidateがより整理され、SSGとISRを大きく区別せずまとめて**Static Rendering**・従来のSSR相当を**Dynamic Rendering**と呼称する形で再定義されました。

| レンダリング                                                                                                                   | タイミング            | Pages Routerとの比較 |
| ------------------------------------------------------------------------------------------------------------------------------ | --------------------- | -------------------- |
| [Static Rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default) | build時やrevalidate後 | SSG・ISR相当         |
| [Dynamic Rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)       | ユーザーリクエスト時  | SSR相当              |

:::message
Server Componentsは[Soft Navigation](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#5-soft-navigation)時も実行されるのでSSR・SSG・ISRと単純に比較できるものではないですが、ここでは簡略化して比較しています。
:::

App Routerは**デフォルトでStatic Rendering**となっており、**Dynamic Renderingはオプトイン**になっています。Dynamic Renderingにオプトインする方法は以下の通りです。

### dynamic functions

`cookies()`/`headers()`などの[dynamic functions](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-functions)と呼ばれる関数を呼び出すと、Dynamic Renderingとなります。

```ts
// page.tsx
import { cookies } from "next/headers";

export default function Page() {
  const cookieStore = cookies();
  const sessionId = cookieStore.get("session-id");

  return "...";
}
```

:::message
[dynamic routes](https://nextjs.org/docs/app/building-your-application/routing/dynamic-routes)における[`searchParams` props](https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional)は、関数ではないですがdynamic functionsの1つとして数えられており、参照するとDynamic Renderingになります。一方[`params` props](https://nextjs.org/docs/app/api-reference/file-conventions/page#params-optional)は、参照するとデフォルトでDynamic Renderingになりますが、[generateStaticParams()](https://nextjs.org/docs/app/api-reference/functions/generate-static-params)を利用するなどするとStatic Renderingになるため、必ずしもDynamic Renderingとは限りません。
:::

### `no-store`な`fetch()`

`fetch()`のオプションで[Data Cache](https://nextjs.org/docs/app/building-your-application/caching#data-cache)をオプトアウトした場合、Dynamic Renderingとなります。キャッシュをオプトアウトするには`fetch()`の第二引数のオプションで`cache: "no-store"`か`next: { revalidate: 0 }`を指定する必要があります。

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
従来`cache`オプションのデフォルトは`"force-cache"`でした。`v15.0.0-rc.0`では、デフォルトが`"no-store"`に変更されていますが、明示的に指定しないとDynamic Renderingにならないという仕様になっています。

これらの仕様については[RC中に変更される可能性](https://x.com/feedthejim/status/1794778189354705190)が示唆されていますが、本書執筆時点では変更されるかは不明です。
:::

### Route Segment Config

[Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)を利用してDynamic Renderingに切り替えることもできます。具体的には、`page.tsx`や`layout.tsx`に以下どちらかを設定することでDynamic Renderingを強制できます。

```tsx
// layout.tsx | page.tsx
export const dynamic = "force-dynamic";
```

```tsx
// layout.tsx | page.tsx
export const revalidate = 0; // 1以上でStatic Rendering
```

:::message
`layout.tsx`に設定したRoute Segment ConfigはLayoutが利用される下層ページにも適用されるため、注意しましょう。
:::

### `unstable_noStore()`

末端のコンポーネントでDynamic Renderingを強制したいがdynamic functionsや`no-store`な`fetch()`を使っていない場合には、`unstable_noStore()`を呼び出すことでDynamic Renderingに切り替えることができます。具体的には、[Prisma](https://www.prisma.io/)を使ったDBアクセス時などに有用でしょう。

```ts
import { unstable_noStore as noStore } from "next/data-store";

export function LeafComponent() {
  noStore();

  // DBアクセスなど

  return "...";
}
```

## 設計・プラクティス

Static Renderingは耐障害性・パフォーマンスに優れています。ユーザーリクエスト毎にレンダリングが必要なら前述の方法でDynamic Renderingにオプトインする必要がありますが、それ以外のケースについて、App Routerでは**可能な限りStatic Renderingにする**ことが推奨されています。

Static Renderingのレンダリング結果であるhtmlやRSC Payloadのキャッシュは、[Full Route Cache](https://nextjs.org/docs/app/building-your-application/caching#full-route-cache)と呼ばれています。App RouterではStatic Renderingを活用するために、Full Route Cacheのオンデマンドrevalidateや時間ベースでのrevalidateといったよくあるユースケースをフォローし、従来のSSGのように変更があるたびにデプロイが必要といったことがないように設計されています。

### オンデマンドrevalidate

`revalidatePath()`や`revalidateTag()`を[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)や[Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)で呼び出すことで、関連するData CacheやFull Route Cacheをrevalidateすることができます。

```ts
"use server";

import { revalidatePath } from "next/cache";

export async function action() {
  // ...

  revalidatePath("/products");
}
```

### 時間ベースrevalidate

Route Segment Configの[revalidate](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#revalidate)を指定することでFull Route Cacheや関連するData Cacheを時間ベースでrevalidateすることができます。

```tsx
// layout.tsx | page.tsx
export const revalidate = 10; // 10s
```

:::message
重複になりますが、`layout.tsx`に`revalidate`を設定するとLayoutが利用される下層ページにも適用されるため、注意しましょう。
:::

例えば1秒などの非常に短い時間でも設定すれば、瞬間的に非常に多くのリクエストが発生したとしてもレンダリングは1回で済むため、バックエンドAPIへの負荷軽減や安定したパフォーマンスに繋がります。更新頻度が非常に高いページでもユーザー間で共有できる(=ユーザー固有の情報などを含まない)のであれば、設定を検討しましょう。

## トレードオフ

### 予期せぬDynamic Renderingとパフォーマンス劣化

Route Segment Configや`unstable_noStore()`によってDynamic Renderingを利用する場合、開発者は明らかにDynamic Renderingを意識して使うのでこれらが及ぼす影響を見誤ることは少ないと考えられます。一方、dynamic functionsは「cookieを利用したい」、`cache: "no-store"`な`fetch`は「Data Cacheを使いたくない」などの主目的が別にあり、これに伴って副次的にDynamic Renderingに切り替わるため、開発者は影響範囲に注意する必要があります。

特に、Data Cacheなどを適切に設定できていないとDynamic Renderingに切り替わった際にページ全体のパフォーマンス劣化につながる可能性があります。こちらについての詳細は後述の[_Dynamic RenderingとData Cache_](part_3_dynamic_rendering_data_cache)をご参照ください。

### レンダリング境界とPPR

従来Dynamic RenderingはRoute単位(`page.tsx`や`layout.tsx`)でしか切り替えることができませんでしたが、v14以降experimentalフラグで**PPR**(Partial Pre-Rendering)を有効にするにより、文字通りPartial(部分的)にDynamic Renderingへの切り替えが可能になります。PPRではStatic/Dynamic Renderingの境界を`<Suspense>`によって定義します。

```tsx
import { Suspense } from "react";
import { PostFeed, Weather } from "./Components";

export default function Posts() {
  return (
    <section>
      {/* `<PostFeed />`はStatic Rendering */}
      <PostFeed />
      <Suspense fallback={<p>Loading weather...</p>}>
        {/* `<Weather />`はDynamic Rendering */}
        <Weather />
      </Suspense>
    </section>
  );
}
```

PPRについては後述の[_Partial Pre Rendering(PPR)_](part_4_partial_pre_rendering)や筆者の過去記事である以下をご参照ください。

https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering
