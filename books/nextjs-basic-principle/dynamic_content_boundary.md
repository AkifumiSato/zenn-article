---
title: "Dynamic Contentの境界"
---

## 要約

Cache ComponentsはPPRをベースとしており、1つのRouteにStatic Shell・Dynamic Content・Cached Data（Component）が含まれます。Route内でDynamic Contentの境界を扱うには、`loading.tsx`か`<Suspense>`境界が必要です。

`<Suspense>`の乱用は[ポップコーンUI](#ポップコーンui)を引き起こすので、慎重に設計しましょう。

## 背景

Cache ComponentsのRouteは以下3つで構成されます。

- **Static Shell**: 事前レンダリングされた断片的なHTMLやRSC Payload
- **Dynamic Content**: リクエスト毎にレンダリングされる`<Suspense>`境界の内側
- **Cached Data（Component）**: Cacheされたデータやコンポーネント

### Static Shell

**Static Shell**は事前レンダリングされた断片的なHTMLやRSC Payloadです。Next.jsはリクエストを受け取り次第、即座にStatic Shellをブラウザに配信します。

事前レンダリング中に`"use cache"`が宣言された関数やコンポーネントが参照された場合、関数の戻り値やレンダリング結果がStatic Shellに含まれます。また、Static ShellにはDynamic Contentの`fallback`が含まれており、Dynamic Content自体はレンダリング完了次第配信されます。

### Dynamic Content

Dynamic Contentはネットワーク通信などを伴う動的な処理や、[Runtime Data↗︎](https://nextjs.org/docs/app/getting-started/cache-components#runtime-data)、[非決定的操作↗︎](https://nextjs.org/docs/app/getting-started/cache-components#non-deterministic-operations)を扱うことができる、`<Suspense>`境界の内側を指します。

:::message
ただし、事前レンダリング時に解決されうるような動的処理はStatic Shell内でも扱うことが可能です。具体的には以下などが挙げられます。

- `fs.readFileSync(path)`
- `await import(module)`
- `JSON.parse(content)`

:::

### Cached Data（Component）

`"use cache"`が宣言された関数やコンポーネントはStatic Shellに含まれる以外にも、Dynamic Content内から参照することが可能です。この場合`"use cache"`は、デフォルトではインメモリCacheのマーカーとして機能します。

## 設計・プラクティス

Dynamic Contentは`<Suspense>`によって境界付けられます。Routeに`loading.tsx`が存在する場合は暗黙的に`<Suspense>`境界が設定されるので、Cache Componentsでは積極的に`loading.tsx`を活用しましょう。

### `loading.tsx`の活用

Cache ComponentsはDynamic IOが含まれているため、`<Suspense>`境界の内側でしか動的I/O処理を扱うことができません。Routeに`loading.tsx`が存在する場合には暗黙的に`<Suspense>`境界が設定されるため、Page側で明示的な`<Suspense>`なしに動的I/O処理を扱うことができます。

以下はブログ記事ページの実装例です。

```tsx
// loading.tsx
export default function Loading() {
  return <div>Loading...</div>;
}

// page.tsx: Dynamic Content
export default async function PostPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  // `loading.tsx`があるので`await params`を扱える
  // `loading.tsx`がなければエラーになる
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

ただし、`layout.tsx`は[下層Routeと共有](#下層routeと共有されるlayouttsx)されることに注意が必要です。

### PageやLayoutの一部のみが動的なRoute

PageやLayoutの一部のみが動的な場合には、明示的に`<Suspense>`境界を追加しましょう。

先ほどのブログ記事ページの実装例において、`generateStaticParams()`を使って全ての`id`を事前に取得することができるとします^[現実的にbuild時解決が可能な量のデータである前提です。]。この場合、Pageに含まれるブログ記事の本文は一般的にCache可能である可能性が高いですが、記事のコメントはリアルタイム性が高いためCacheすべきでないと考えられます。

```tsx
// page.tsx: Static Shell+Dynamic Content
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

このように、事前レンダリング時に大部分は生成可能だが一部のみが動的なRouteにおいては、PageやLayout内で明示的に`<Suspense>`境界を追加することを推奨します。

### パフォーマンスチューニング

パフォーマンスチューニングの観点から、明示的な`<Suspense>`境界を追加することが有用となることがあります。一部のコンポーネントが非常に低速なデータフェッチを含むなどしてパフォーマンスボトルネックである場合、`<Suspense>`境界を追加することでクライアントへの送信が分割されるため、パフォーマンスが改善する可能性があります。

```tsx
// loading.tsx
export default function Loading() {
  return <div>Loading...</div>;
}

// page.tsx: Dynamic Content
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

ただし、`<Suspense>`境界を多用しStreaming配信を分割しすぎると、[ポップコーンUI](#ポップコーンui)を引き起こす可能性があるため、慎重に設計しましょう。

## トレードオフ

### 下層Routeと共有される`layout.tsx`

前述の通り、LayoutはRouteにおけるStatic Shell部分とみなすことが可能ですが、Layoutは[Nesting Layouts↗︎](https://nextjs.org/docs/app/getting-started/layouts-and-pages#nesting-layouts)なため、下層Routeと共有されることに注意が必要です。

```
./app/products/[id]/
├── layout.tsx // `app/products/[id]/reviews`とも共有される
├── page.tsx
└── reviews/
    ├── layout.tsx
    └── page.tsx
```

この問題に対する対処法は2つ考えられます。

#### Route Groupsの活用

[Route Groups↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/route-groups)を使って、共有レイアウトとRoute固有レイアウトを分けることができます。以下は商品ページと商品レビューページそれぞれで固有の`layout.tsx`を設定する例です。

```
./app/products/[id]/
├── layout.tsx // 共有Layout
├── (index)/
│     ├── layout.tsx // `(index)`固有のLayout
│     └── page.tsx
└── reviews/
    ├── layout.tsx // `reviews`固有のLayout
    └── page.tsx
```

#### 明示的な`<Suspense>`境界

Layoutは共有レイアウトのみを記述する場所と割り切り、Page内に`<Suspense>`境界を明示的に追加することも戦略として考えられます。

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

ただし、`<Suspense>`境界は初学者やチームの新規参画者が躓くポイントにもなりやすいので、筆者はRoute Groupsの活用から検討することを推奨します。

### ポップコーンUI

**ポップコーンUI**は、画面内の要素が徐々に表示されるようなUIのことを指し、ポップコーンが弾ける様子を揶揄したものです。このような体験は、ユーザーにとって不快な体験となります。

`<Suspense>`を乱用するとポップコーンUIを引き起こす可能性があるので、ユーザーにとって意味のあるUIのまとまり単位で`<Suspense>`境界を設計するよう意識しましょう。

### Cumulative Layout Shift

**Cumulative Layout Shift**（CLS）は[Core Web Vitals↗︎](https://web.dev/articles/vitals?hl=ja#core-web-vitals)の一つで、ページ読み込み中に発生する予期せぬレイアウトの変化を数値化した指標です。Static Shellに`fallback`が含まれるということは、読み込み中にレイアウトが変化する可能性をトレードオフすることでもあります。

`<Suspense>`境界はLayout Shiftを意識して、スケルトンUIの活用や配信順を適切に設計しましょう。
