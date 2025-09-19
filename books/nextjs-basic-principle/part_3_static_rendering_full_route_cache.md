---
title: "Static RenderingとFull Route Cache"
---

## 要約

Static Renderingでは、HTMLやRSC PayloadのキャッシュであるFull Route Cacheを生成します。Full Route Cacheは短い期間でrevalidate可能なので、ユーザー固有の情報を含まないようなページは積極的にFull Route Cacheを活用しましょう。

## 背景

Next.jsは、従来Pages Routerで[SSR↗︎](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG↗︎](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)という3つのレンダリングモデル^[[こちらの記事↗︎](https://vercel.com/blog/partial-prerendering-with-next-js-creating-a-new-default-rendering-model)より引用。レンダリング戦略とも呼ばれます。]をサポートしてきました。App Routerでは上記相当のレンダリングをサポートしつつ、revalidateがより整理され、SSGとISRを大きく区別せずまとめて**Static Rendering**、従来のSSR相当を**Dynamic Rendering**と呼称する形で[サーバー側レンダリングが再定義↗︎](https://nextjs.org/docs/app/getting-started/linking-and-navigating#server-rendering)されました。

| レンダリング          | タイミング            | Pages Routerとの比較 |
| --------------------- | --------------------- | -------------------- |
| **Static Rendering**  | build時やrevalidate後 | SSG・ISR相当         |
| **Dynamic Rendering** | ユーザーリクエスト時  | SSR相当              |

:::message
Server Componentsは[Soft Navigation↗︎](https://nextjs.org/docs/app/getting-started/linking-and-navigating#client-side-transitions)時も実行されるのでSSR・SSG・ISRと単純に比較できるものではないですが、ここでは簡略化して比較しています。
:::

App Routerは**デフォルトでStatic Rendering**となっており、**Dynamic Renderingはオプトイン**になっています。Dynamic Renderingにオプトインする方法は以下の通りです。

### Dynamic APIs

`cookies()`/`headers()`などの[Dynamic APIs↗︎](https://nextjs.org/docs/app/guides/caching#dynamic-apis)と呼ばれるAPIを利用すると、Dynamic Renderingとなります。

```ts
// page.tsx
import { cookies } from "next/headers";

export default async function Page() {
  const cookieStore = await cookies();
  const sessionId = cookieStore.get("session-id");

  return "...";
}
```

:::message
[Dynamic Routes↗︎](https://nextjs.org/docs/app/getting-started/project-structure#dynamic-routes)における[`searchParams` props↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional)はDynamic APIsの1つとして数えられており、参照するとDynamic Renderingになります。一方[`params` props↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/page#params-optional)は、参照するとデフォルトでDynamic Renderingになりますが、[generateStaticParams()↗︎](https://nextjs.org/docs/app/api-reference/functions/generate-static-params)を利用するなどするとStatic Renderingになるため、必ずしもDynamic Renderingになるとは限りません。
:::

### `cache: "no-store"`もしくは`next.revalidate: 0`な`fetch()`

[`fetch()`のオプション↗︎](https://nextjs.org/docs/app/api-reference/functions/fetch#optionscache)で`cache: "no-store"`を指定した場合や、`next: { revalidate: 0 }`を指定した場合、Dynamic Renderingとなります。

:::message alert
v14以前において、[`cache`オプション↗︎](https://nextjs.org/docs/app/api-reference/functions/fetch#optionscache)のデフォルトは`"force-cache"`でした。v15ではデフォルトでキャッシュが無効になるよう変更されていますが、デフォルトではStatic Renderingとなっています。Dynamic Renderingに切り替えるには明示的に`"no-store"`を指定する必要があるので、注意しましょう。
:::

```ts
// page.tsx
export default async function Page() {
  const res = await fetch("https://dummyjson.com/todos/random", {
    // 🚨Dynamic Renderingにするために`"no-store"`を明示
    cache: "no-store",
  });
  const todoItem: TodoItem = await res.json();

  return "...";
}
```

### Route Segment Config

[Route Segment Config↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)を利用してDynamic Renderingに切り替えることもできます。具体的には、`page.tsx`や`layout.tsx`に以下どちらかを設定することでDynamic Renderingを強制できます。

```tsx
// layout.tsx | page.tsx
export const dynamic = "force-dynamic";
```

```tsx
// layout.tsx | page.tsx
export const revalidate = 0; // 1以上でStatic Rendering
```

:::message alert
`layout.tsx`に設定したRoute Segment ConfigはLayoutが利用される下層ページにも適用されるため、注意しましょう。
:::

### `connection()`

末端のコンポーネントから利用者側にDynamic Renderingを強制したいが、`headers()`や`no-store`な`fetch()`を使っていない場合には、[`connection()`↗︎](https://nextjs.org/docs/app/api-reference/functions/connection)でDynamic Renderingに切り替えることができます。具体的には、[Prisma↗︎](https://www.prisma.io/)を使ったDBアクセス時などに有用でしょう。

```ts
import { connection } from "next/server";

export async function LeafComponent() {
  await connection();

  // DBアクセスなど

  return "...";
}
```

## 設計・プラクティス

Static Renderingは耐障害性・パフォーマンスに優れています。ユーザーリクエスト毎にレンダリングが必要なら前述の方法でDynamic Renderingにオプトインする必要がありますが、それ以外のケースについて、Next.jsでは**可能な限りStatic Renderingにする**ことが推奨されています。

Static Renderingのレンダリング結果であるHTMLやRSC Payloadのキャッシュは、[Full Route Cache↗︎](https://nextjs.org/docs/app/guides/caching#full-route-cache)と呼ばれています。Next.jsではStatic Renderingを活用するために、Full Route Cacheのオンデマンドrevalidateや時間ベースでのrevalidateといったよくあるユースケースをフォローし、SSGのように変更があるたびにデプロイが必要といったことがないように設計されています。

### オンデマンドrevalidate

[`revalidatePath()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)や[`revalidateTag()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)をServer Actions^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]や[Route Handlers↗︎](https://nextjs.org/docs/app/getting-started/route-handlers-and-middleware#route-handlers)で呼び出すことで、関連するData CacheやFull Route Cacheをrevalidateすることができます。

```ts
"use server";

import { revalidatePath } from "next/cache";

export async function action() {
  // ...

  revalidatePath("/products");
}
```

### 時間ベースrevalidate

Route Segment Configの[revalidate↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#revalidate)を指定することでFull Route Cacheや関連するData Cacheを時間ベースでrevalidateすることができます。

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

Route Segment Configや`connection()`によってDynamic Renderingを利用する場合、開発者は明らかにDynamic Renderingを意識して使うのでこれらが及ぼす影響を見誤ることは少ないと考えられます。一方、Dynamic APIsは「cookieを利用したい」、`cache: "no-store"`な`fetch`は「Data Cacheを使いたくない」などの主目的が別にあり、これに伴って副次的にDynamic Renderingに切り替わるため、開発者は影響範囲に注意する必要があります。

特に、Data Cacheなどを適切に設定できていないとDynamic Renderingに切り替わった際にページ全体のパフォーマンス劣化につながる可能性があります。こちらについての詳細は後述の[Dynamic RenderingとData Cache](part_3_dynamic_rendering_data_cache)をご参照ください。
