---
title: "Dynamic Contentの境界"
---

## 要約

Cache ComponentsはPPRをベースとしており、1つのRouteにStatic Shell・Dynamic Content・Cached Data（Component）が含まれます。Route内でDynamic Contentの境界を扱うには、`loading.tsx`か`<Suspense>`境界が必要です。

`<Suspense>`の乱用は[ポップコーンUI](#ポップコーンui)を引き起こすので、慎重に設計しましょう。

## 背景

Cache ComponentsのRouteは以下3つで構成されます。

- **Static Shell**: 事前レンダリングされたHTMLやRSC Payloadの外郭
- **Dynamic Content**: リクエスト毎にレンダリングされる`<Suspense>`境界の内側
- **Cached Data（Component）**: Cacheされたデータやコンポーネント

### Static Shell

**Static Shell**は事前レンダリングされたHTMLやRSC Payloadの外郭です。Next.jsはリクエストを受け取り次第、即座にStatic Shellをブラウザに配信します。

事前レンダリング中`"use cache"`によりCache可能とされた関数やコンポーネントが参照された場合、関数の戻り値やレンダリング結果がStatic Shellに含まれます。また、Static ShellにはDynamic Contentの`fallback`が含まれており、Dynamic Content自体はレンダリング完了次第chunkが配信されます。

### Dynamic Content

Dynamic Contentはネットワーク通信などを伴う動的な処理や、[Runtime Data↗︎](https://nextjs.org/docs/app/getting-started/cache-components#runtime-data)、[非決定的操作↗︎](https://nextjs.org/docs/app/getting-started/cache-components#non-deterministic-operations)を扱うような`<Suspense>`境界の内側^[[事前レンダリング時に解決されうるような動的処理↗︎](https://nextjs.org/docs/app/getting-started/cache-components#automatically-prerendered-content)はStatic Shell内でも扱うことが可能です]のコンテンツを指します。

### Cached Data（Component）

`"use cache"`が宣言された関数やコンポーネントはStatic Shellに含まれる以外にも、Dynamic Content内から参照することが可能です。この場合`"use cache"`は、デフォルトではインメモリCacheのマーカーとして機能します。

## 設計・プラクティス

Dynamic Contentは`<Suspense>`や`loading.tsx`によって境界付けられます。Cache Componentsでは積極的に`loading.tsx`を活用しましょう。

### `loading.tsx`の活用

Routeに`loading.tsx`が存在する場合には暗黙的に`<Suspense>`境界が設定されるため、Page側で明示的な`<Suspense>`なしに動的I/O処理を扱うことができます。

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
  // ✅`loading.tsx`があるので`await params`を扱える
  // ⚠️`loading.tsx`がなければエラーになる
  const { id } = await params;

  return (
    <>
      <PostContainer id={id} />
      <CommentsContainerContainer id={id} />
    </>
  );
}
```

この場合Layoutは`<Suspense>`境界の外側になるので、以下のような責務分けと考えることができます。

- `layout.tsx`: RouteにおけるStatic Shell部分
- `loading.tsx`: Fallback UI
- `page.tsx`: Dynamic Content

ただし、`layout.tsx`は[下層Routeと共有](#下層routeと共有されるlayouttsx)されることに注意が必要です。

```
./app/
├── layout.tsx          // <AppLayout>
└── posts/
    ├── layout.tsx      // <PostsLayout>
    └── [id]/
        ├── layout.tsx  // <PostIdLayout>
        ├── loading.tsx // <Loading>
        └── page.tsx    // <Page>
```

```tsx
<AppLayout>
  <PostsLayout>
    <PostIdLayout>
      <Suspense fallback={<Loading />}>
        <Page />
      </Suspense>
    </PostIdLayout>
  </PostsLayout>
</AppLayout>
```

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
      {/* <PostContainer>: Static Rendering */}
      <PostContainer id={id} />
      <Suspense fallback={<Loading />}>
        {/* <CommentsContainer>: Dynamic Rendering */}
        <CommentsContainer id={id} />
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

パフォーマンスチューニングの観点から、明示的な`<Suspense>`境界を追加することが有用となることがあります。一部のコンポーネントが非常に低速なデータフェッチを含むなどしてパフォーマンスボトルネックである場合、`<Suspense>`境界を追加することでクライアントへのchunk送信が分割されるため、パフォーマンスが改善する可能性があります。

以下は商品ページにおいて、おすすめ商品の表示がボトルネックで遅延させる実装例です。

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
      <ProductDetailsContainer id={id} />
      <ProductReviewsContainer id={id} />
      <Suspense fallback={<Loading />}>
        {/* おすすめ商品の取得が非常に低速なので、chunkを分割 */}
        <RecommendationsContainer id={id} />
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
      {/* <ProductDetailContainer>: Static Rendering */}
      <ProductDetailContainer id={id} />
      {/* <ProductReviewsContainer>: Static Rendering */}
      <ProductReviewsContainer id={id} />
      <Suspense fallback={<Loading />}>
        {/* <RecommendationsContainer>: Dynamic Rendering */}
        <RecommendationsContainer id={id} />
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
