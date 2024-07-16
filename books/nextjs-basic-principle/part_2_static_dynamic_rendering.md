---
title: "static/dynamic rendering"
---

## 要約

static/dynamic renderingを強く意識し、可能な限りstatic renderingにしましょう。

## 背景

従来Pages Routerではサーバー側のレンダリングについて、[SSR](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)という3つのレンダリングモデルをサポートしてきました。App Routerではこれら同等のレンダリングが可能ですが、[`revalidatePath()`](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)などのAPIによって任意のタイミングでrevalidateできるためにSSGとISRを大別する必要がなくなり、これらをまとめて**static rendering**、従来のSSR相当を**dynamic rendering**と呼称する形で再整理されました。

- [static rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default): 従来のSSGやISR相当で、build時やrevalidate実行後にレンダリング
  - revalidateなし: SSG相当
  - revalidateあり: ISR相当
- [dynamic rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering): 従来のSSR相当で、リクエストごとにレンダリング

revalidateについては後述の[Full Route Cache/Data Cache](part_2_server_cache)や[データ操作とServer Actions](part_2_data_mutation)にて詳細に解説します。

App Routerは**デフォルトでstatic rendering**となっており、**dynamic renderingはオプトイン**になっています。dynamic renderingにオプトインする方法は以下の通りです。

### dynamic functions

`cookies()`/`headers()`などの[dynamic functions](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-functions)と呼ばれる関数を呼び出すと、dynamic renderingとなります。

```ts
// app/page.tsx
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
// app/products/[id]/page.tsx
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

[Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)を利用してdynamic renderingに切り替えることもできます。具体的には、`page.tsx`や`layout.tsx`に`dynamic`か`revalidate`を以下のように設定することでdynamic renderingを強制できます。

```tsx
// app/page.tsx
export const dynamic = "force-dynamic";
```

```tsx
// app/page.tsx
export const revalidate = 0; // 1以上でstatic rendering
```

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

static renderingは耐障害性・パフォーマンスに優れるため、App Routerにおいては**可能な限りstatic renderingにすることが望ましい**と言えます。短い期間でもキャッシュ可能ならstatic renderingにすべきです。完全にユーザーリクエスト毎にレンダリングが必要ならdynamic renderingにしましょう。

### static renderingのrevalidate

ユーザー情報を含む場合などキャッシュが完全に不可能なページはdynamic renderingであるべきですが、それ以外については大きく分けて以下3つのパターンが考えられます。

- 完全に静的なページ
- 定期的な更新が必要なページ
- 任意のタイミングで更新が必要なページ

「完全に静的なページ」は当然ながらstatic renderingで問題ありません。

「定期的な更新が必要なページ」はRoute Segment Configの[revalidate](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#revalidate)を指定することでstatic renderingにしつつ定期的に再レンダリングすることができます。

「任意のタイミングで更新が必要なページ」は前述の[`revalidatePath()`](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)や[`revalidateTag()`](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)を呼び出すことでstatic renderingにしつつ任意のタイミングで再レンダリングすることができます。

### PPR

Next.jsの`v15.0.0-rc.0`では、experimentalですが**PPR**(Partial Pre-Rendering)を利用することができます。従来、ページ単位でしかdynamic renderingへ切り替えられなかったのが、PPRでは文字通りPartial(部分的)に切り替えが可能になります。static/dynamic renderingの境界は、`<Suspense>`によって定義できます。

PPRの詳細は筆者の過去記事をご参照ください。

https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering

## トレードオフ

### 予期せぬdynamic renderingとパフォーマンス劣化

Route Segment Configや`unstable_noStore()`によってdynamic renderingを利用する場合、開発者は明らかにdynamic renderingを意識して使うのでこれらが及ぼす影響を見誤ることは少ないと考えられます。一方、dynamic functionsや`cache: "no-store"`な`fetch`は主たる目的が別にあり、副次的にdynamic renderingに切り替わるため、これらを利用する際の影響範囲を開発者がしっかり意識する必要があります。

特に、Data Cacheなどを適切に設定できていないとdynamic renderingに切り替わった際にページ全体のパフォーマンス劣化につながる可能性があります。`fetch`を用いた外部APIアクセスであればデフォルトで強くData Cacheが利用されますが、DBアクセスやGraphQLにおいては、自身でData Cacheの設定・実装が必要になるので特に注意が必要です。
