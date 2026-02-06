---
title: "Dynamic ContentとSuspense"
---

## 要約

Cache ComponentsにはPPRが含まれており、RouteはStatic Shell・Dynamic Content・Cached Data(Component)で構成されます。Route内でDynamic Renderingを扱うには、`loading.tsx`か`<Suspense>`境界が必要です。

`<Suspense>`の乱用は[ポップコーンUI](#ポップコーンui)を引き起こすので、慎重に設計しましょう。

## 背景

Cache ComponentsにはPPRが含まれており、全てのRouteはPre-Renderingされます。そのため、Cache Componentsでは1つのRouteをStatic Shell・Dynamic Content・Cached Data(Component)に分解して考えることができます。

- **Static Shell**: Pre-Renderingされた断片的なHTMLやRSC Payload
- **Dynamic Content**: リクエスト毎にレンダリングされる動的なコンポーネント境界
- **Cached Data(Component)**: Cacheされたデータやコンポーネント

### Static Shell

**Static Shell**はPre-Renderingされた断片的なHTMLやRSC Payloadです。Next.jsはRouteのリクエストを受け取り次第、即座にStatic Shellをブラウザに配信します。

Pre-Rendering中に`"use cache"`が宣言された関数やコンポーネントが参照された場合、関数の戻り値やレンダリング結果がStatic Shellに含まれます。また、Static ShellにはDynamic Contentの`fallback`が含まれており、Dynamic Content自体はレンダリング完了次第遅れて配信されます。

### Dynamic Content

Dynamic Contentはネットワーク通信などを伴う動的な処理や、[Runtime Data↗︎](https://nextjs.org/docs/app/getting-started/cache-components#runtime-data)、[非決定的操作↗︎](https://nextjs.org/docs/app/getting-started/cache-components#non-deterministic-operations)を扱うことができる、`<Suspense>`境界内のコンポーネント群を指します。

:::message
ただし、Pre-Rendering時に解決されうるような動的処理はStatic Shell内でも扱うことが可能です。具体的には以下などが挙げられます。

- `fs.readFileSync(path)`
- `await import(module)`
- `JSON.parse(content)`

:::

### Cached Data(Component)

`"use cache"`が宣言された関数やコンポーネントはStatic Shellに含まれる以外にも、Dynamic Content内から参照することが可能です。この場合`"use cache"`は、デフォルトではインメモリCacheのマーカーとして機能します。

:::message
Vercelでは意図して、Dynamic Content内における`"use cache"`をCacheしないことがあるようです。詳細な仕様や背景については以下issue↗の解説をご参照ください。

https://github.com/vercel/next.js/issues/85240#issuecomment-3560124078
:::

## 設計・プラクティス

Cache ComponentsでDynamic Contentを扱うには、`<Suspense>`境界が必要です。Routeに`loading.tsx`が存在する場合、Next.jsはPageに対し`<Suspense>`を設定するので、Cache Componentsでは積極的に`loading.tsx`を活用しましょう。

### `loading.tsx`の活用

`loading.tsx`が設定されている場合、Next.jsは`page.tsx`に対し`<Suspense>`境界を追加するため、`page.tsx`で動的処理を扱うことができます。以下はブログ記事ページの実装例です。

```tsx
// loading.tsx
export default function Loading() {
  return <div>Loading...</div>;
}

// page.tsx
export default async function PostPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  // `loading.tsx`があるので`await params`を扱える
  const { id } = await params;

  return (
    <>
      <PostContent id={id} />
      <Comments id={id} />
    </>
  );
}
```

この場合`layout.tsx`は`<Suspense>`境界の外側になるので、以下のような責務分けと考えることができます。

- `layout.tsx`: RouteにおけるStatic Shell部分
- `loading.tsx`: Fallback UI
- `page.tsx`: Dynamic Content

Next.js側の実装イメージとしては以下のようなツリー構造になります。

```tsx
// <Layout>: layout.tsx
// <Loading>: loading.tsx
// <Page>: page.tsx
<Layout>
  <Suspense fallback={<Loading />}>
    <Page />
  </Suspense>
</Layout>
```

このように、Cache Componentsでは静的部分を`layout.tsx`に、動的部分を`page.tsx`に実装することが可能になります。ただし、`layout.tsx`は[下層Routeと共有](#下層routeと共有されるlayouttsx)されることに注意が必要です。

### `page.tsx`や`layout.tsx`の一部のみが動的なRoute

`page.tsx`や`layout.tsx`の一部のみが動的なRouteでは、明示的に`<Suspense>`境界を追加しましょう。

先ほどのブログ記事ページの実装例において、`generateStaticParams()`を使って全ての`id`を事前に取得することができます^[現実的にbuild時解決が可能な量のデータである前提です。]。この場合`page.tsx`に含まれるブログ記事の本文は一般的にCache可能である可能性が高いですが、記事のコメントはリアルタイム性が高いためCacheすべきでないと考えられます。

```tsx
export default async function PostPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  // generateStaticParamsしてるのでbuild時に解決される
  const { id } = await params;

  return (
    <>
      {/* <PostContent>: Static Rendering */}
      <PostContent id={id} />
      <Suspense fallback={<Loading />}>
        {/* <Comments>: Dynamic Rendering */}
        <Comments id={id} />
      </Suspense>
    </>
  );
}

export async function generateStaticParams() {
  const { posts } = await getPosts();
  return posts.map((post) => ({ id: post.id.toString() }));
}
```

このように、build時に大部分は生成可能だが一部のみが動的なRouteにおいては、`page.tsx`や`layout.tsx`内で明示的に`<Suspense>`境界を追加することが有用となります。

### パフォーマンスチューニング

パフォーマンスチューニングの観点から、明示的な`<Suspense>`境界を追加することが有用となることがあります。一部のコンポーネントが非常に低速なデータフェッチを含むなどでパフォーマンスボトルネックな場合、`<Suspense>`境界を追加することでクライアントへの送信が分割されるため、パフォーマンスが改善する可能性があります。

```tsx
// loading.tsx
export default function Loading() {
  return <div>Loading...</div>;
}

// page.tsx
export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  return (
    <>
      <ProductDetails id={id} />
      <ProductReviews id={id} />
      <Suspense fallback={<Loading />}>
        {/* <ProductRecommendations>: おすすめ商品の取得が非常に低速なので、Streaming配信を分割 */}
        <ProductRecommendations id={id} />
      </Suspense>
    </>
  );
}
```

ただし、`<Suspense>`境界を乱用しStreaming配信を分割しすぎると、[ポップコーンUI](#ポップコーンui)を引き起こす可能性があるため、慎重に設計しましょう。

## トレードオフ

### 下層Routeと共有される`layout.tsx`

前述の通り、`layout.tsx`は「RouteにおけるStatic Shell部分」とみなすことが可能ですが、`layout.tsx`は[Nesting Layouts↗︎](https://nextjs.org/docs/app/getting-started/layouts-and-pages#nesting-layouts)なため、下層Routeと共有されることに注意が必要です。この問題に対する対処法は2つ考えられます。

#### Route Groupsの活用

[Route Groups↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/route-groups)を使って共有レイアウトの`layout.tsx`とRoute固有レイアウトの`layout.tsx`を分けることで、`layout.tsx`の共有を防ぐことができます。

以下は商品ページと商品レビューページそれぞれで固有の`layout.tsx`を設定する例です。

```
app/products/[id]/
├── (index)/
│     ├── layout.tsx
│     └── page.tsx
└── reviews/
    ├── layout.tsx
    └── page.tsx
```

他の手段を検討する前に、まずはRoute Groupsを活用することを検討しましょう。

#### 明示的な`<Suspense>`境界

`layout.tsx`は「RouteにおけるStatic Shell部分」ではなく「共有レイアウトのみを記述する場所」とみなし、Static Shell部分は`page.tsx`内に記述し、動的部分は明示的に`<Suspense>`境界を追加することも戦略として考えられます。

```tsx
// page.tsx
export default function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  // `generateStaticParams()`を使って全ての`id`を事前に取得する
  const { id } = await params;

  return (
    <>
      {/* <ProductDetails>: Static Rendering */}
      <ProductDetails id={id} />
      {/* <ProductReviews>: Static Rendering */}
      <ProductReviews id={id} />
      <Suspense fallback={<Loading />}>
        {/* <ProductRecommendations>: Dynamic Rendering */}
        <ProductRecommendations id={id} />
      </Suspense>
    </>
  );
}
```

### ポップコーンUI

**ポップコーンUI**は、画面内の要素が徐々に表示されるようなUIのことを指し、ポップコーンが弾ける様子を揶揄したものです。このような体験は、ユーザーにとって不快な体験となります。

`<Suspense>`を乱用すると、ポップコーンUIを引き起こす可能性があるので、慎重に設計しましょう。

### Cumulative Layout Shift

**Cumulative Layout Shift**（CLS）は[Core Web Vitals↗︎](https://web.dev/articles/vitals?hl=ja#core-web-vitals)の一つで、ページ読み込み中に発生する予期せぬレイアウトの変化を数値化した指標です。`<Suspense>`利用時に`fallback`が表示されるということは、読み込み中にレイアウトが変化する可能性をトレードオフすることでもあります。

ページ下部の要素が先に表示されることのないように`<Suspense>`境界を配置することで、Layout Shiftを防ぎましょう。また、スケルトンUIを活用することで体験の悪化を緩和することができます。
