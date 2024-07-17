---
title: "static renderingとFull Route Cache"
---

## 要約

revalidateを駆使して可能な限りstatic renderingにし、Full Route Cacheを活用しましょう。

## 背景

従来Pages Routerではサーバー側のレンダリングについて、[SSR](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)という3つのレンダリングモデルをサポートしてきました。App Routerでは上記同等のレンダリングをサポートしつつ、オンデマンドなrevalidateがより整理されてSSGとISRを大きく区別する意味がなくなったため、これらをまとめて**static rendering**・従来のSSR相当を**dynamic rendering**と呼称する形で再定義されました。

| 種別                                                                                                                           | タイミング            | Pages Routerとの比較 |
| ------------------------------------------------------------------------------------------------------------------------------ | --------------------- | -------------------- |
| [static rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default) | build時やrevalidate後 | SSG・ISR相当         |
| [dynamic rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)       | ユーザーリクエスト時  | SSR相当              |

:::message
Server Componentsは[Soft Navigation](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#5-soft-navigation)時も実行されるので必ずしもSSR・SSG・ISRと比較できるものではないですが、ここでは簡略化して表現しています。
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

### `cache: "no-store"`な`fetch()`

`fetch()`のオプションに`cache: "no-store"`を指定した場合、dynamic renderingとなります。

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

static renderingは耐障害性・パフォーマンスに優れています。完全にユーザーリクエスト毎にレンダリングが必要なら上記いずれかの方法でdynamic renderingにオプトインする必要がありますが、それ以外のケースについてApp Routerでは**可能な限りstatic renderingにする**ことが推奨されています。

static renderingのレンダリング結果のキャッシュは[Full Route Cache](https://nextjs.org/docs/app/building-your-application/caching#full-route-cache)と呼ばれ、定期的なrevalidateもしくはオンデマンドなrevalidateが可能です。これらを駆使して可能な限りstatic renderingにするよう心がけましょう。

### オンデマンドrevalidate

[`revalidatePath()`](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)や[`revalidateTag()`](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)を[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)や[Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)で呼び出すことで、Full Route Cacheを任意のタイミングでrevalidateすることができます。

```ts
"use server";

import { revalidatePath } from "next/cache";

export async function action() {
  // ...

  revalidatePath("/products");
}
```

コメント投稿のようなサイト内からの更新に伴うrevalidateはServer Actionsを、CMS管理画面でのブログ更新のようなサイト外からの更新に伴うrevalidateにはRoute Handlerを組み合わせて利用すると良いでしょう。

これらの詳細は[データ操作とServer Actions](part_2_data_mutation_inner)や[外部で発生したデータ操作](part_2_data_mutation_outer)の章でより詳細に解説します。

### 定期的なrevalidate

Route Segment Configの[revalidate](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#revalidate)を指定することでFull Route Cacheを定期的にrevalidateすることができます。

```tsx
// layout.tsx | page.tsx
export const revalidate = 10; // 10s
```

:::message
重複になりますが、`layout.tsx`に`revalidate`を設定するとLayoutが利用される下層ページにも適用されるため、注意しましょう。
:::

非常に短い時間例えば1秒設定するだけでも、秒間数百のリクエストが発生しても1つにまとめることができるので、バックエンドAPIへの負荷軽減・安定したパフォーマンスを実現できます。

### データの更新頻度から見た使い分け

revalidateは参照するデータの更新頻度に応じて使い分ける必要があります。アプリケーション特性によって様々なケースが考えられますが、大まかに筆者なりの使い分けを以下に示します。

| 更新頻度 | revalidate   | 例                   |
| -------- | ------------ | -------------------- |
| 無       | 無           | LP、規約ページ       |
| 低       | オンデマンド | ブログ記事ページ     |
| 高       | 定期的       | ブログ記事コメント欄 |

## トレードオフ

### 予期せぬdynamic renderingとパフォーマンス劣化

Route Segment Configや`unstable_noStore()`によってdynamic renderingを利用する場合、開発者は明らかにdynamic renderingを意識して使うのでこれらが及ぼす影響を見誤ることは少ないと考えられます。一方、dynamic functionsや`cache: "no-store"`な`fetch`は主たる目的が別にあり、副次的にdynamic renderingに切り替わるため、これらを利用する際の影響範囲を開発者が注意する必要があります。

特に、Data Cacheなどを適切に設定できていないとdynamic renderingに切り替わった際にページ全体のパフォーマンス劣化につながる可能性があります。こちらについての詳細は後述の[dynamic renderingとData Cache](part_2_dynamic_rendering_data_cache)をご参照ください。

### static/dynamic rendering境界とPPR

Next.jsのv14時点では、static/dynamic renderingはRoute単位(`page.tsx`や`layout.tsx`)でしか切り替えられません。

Next.jsのv15(RC.0)では、experimentalですが**PPR**(Partial Pre-Rendering)を利用することができます。従来、Route単位でしかdynamic renderingへ切り替えられなかったのが、PPRでは文字通りPartial(部分的)に切り替えが可能になります。static/dynamic renderingの境界は、`<Suspense>`によって定義できます。

PPRについては後述の[PPRの章](part_3_partial_pre_rendering)や筆者の過去記事である以下をご参照ください。

https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering
