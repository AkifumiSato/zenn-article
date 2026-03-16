---
title: "Dynamic Contentの境界"
---

## 要約

Cache ComponentsはPPRをベースとしており、1つのRouteにStatic Shell・Dynamic Content・Cached Contentが含まれます。Route内でDynamic Contentの境界を扱うには、`loading.tsx`か`<Suspense>`境界が必要です。

`<Suspense>`の乱用は[ポップコーンUI](#ポップコーンui)を引き起こすので、慎重に設計しましょう。

## 背景

Cache ComponentsのRouteは以下3つで構成されます。

- **Static Shell**: 事前レンダリングされた`<Suspense>`境界の外側
- **Dynamic Content**: リクエスト毎にレンダリングされる`<Suspense>`境界の内側
- **Cached Content**: キャッシュされたデータやコンポーネント

### Static Shell

**Static Shell**は事前レンダリングされた`<Suspense>`境界の外側にあるHTMLやRSC Payloadです。Next.jsはリクエストを受け取り次第、即座にStatic Shellをブラウザに配信します。

`"use cache"`によりキャッシュ可能とマークされた関数やコンポーネントが事前レンダリング中に参照されると、それらのレンダリング結果はStatic Shellに含まれます。また、Static ShellにはDynamic Contentのフォールバックが含まれており、Dynamic Content自体はレンダリングが完了次第ブラウザに配信されます。

### Dynamic Content

Dynamic Contentはネットワーク通信などを伴う動的な処理や、[Runtime Data↗︎](https://nextjs.org/docs/app/getting-started/cache-components#runtime-data)、[非決定的操作↗︎](https://nextjs.org/docs/app/getting-started/cache-components#non-deterministic-operations)を扱うような`<Suspense>`境界の内側^[[事前レンダリング時に解決されうるような動的処理↗︎](https://nextjs.org/docs/app/getting-started/cache-components#automatically-prerendered-content)はStatic Shell内でも扱うことが可能です]のコンテンツを指します。

### Cached Content

Cached Contentは`"use cache"`が宣言された関数やコンポーネントで、Static Shellに含まれる以外にもDynamic Content内から参照することが可能です。この場合`"use cache"`は、デフォルトではインメモリキャッシュのマーカーとして機能します。

## 設計・プラクティス

Dynamic Contentは`<Suspense>`や`loading.tsx`によって境界付けられます。Cache Componentsでは積極的に`loading.tsx`を活用しましょう。

### `loading.tsx`の活用

Routeに`loading.tsx`が存在する場合には暗黙的に`<Suspense>`境界が設定されるため、ページ側で明示的な`<Suspense>`なしに動的I/O処理を扱うことができます。

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
      <CommentsContainer id={id} />
    </>
  );
}
```

この場合レイアウトは`<Suspense>`境界の外側になるので、以下のような責務分けと考えることができます。

- `layout.tsx`: RouteにおけるStatic Shell部分
- `loading.tsx`: Fallback UI
- `page.tsx`: Dynamic Content

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

ただし、`layout.tsx`は[#下層Routeと共有](#下層routeと共有されるlayouttsx)されることに注意が必要です。

### ページやレイアウトの一部のみが動的なRoute

ページやレイアウトの一部のみが動的な場合には、明示的に`<Suspense>`境界を追加しましょう。

先ほどのブログ記事ページの実装例において、`generateStaticParams()`を使って全ての`id`を事前に取得することができるとします^[現実的にbuild時解決が可能な量のデータである前提です。]。この場合、ページに含まれるブログ記事の本文は一般的にキャッシュ可能である可能性が高いですが、記事のコメントはリアルタイム性が高いためキャッシュすべきでないと考えられます。

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

このように、大部分は事前レンダリング可能で一部のみが動的なRouteにおいては、ページやレイアウト内で明示的に`<Suspense>`境界を追加することを推奨します。

### パフォーマンスチューニング

パフォーマンスチューニングの観点から、明示的な`<Suspense>`境界を追加することが有用となることがあります。一部のコンポーネントが非常に低速なデータフェッチを含むなどしてパフォーマンスボトルネックである場合、`<Suspense>`境界を追加することでクライアントへ素早くフォールバックが送信されるためUXが改善します。

以下は商品ページにおいて、おすすめ商品の表示を遅延させる実装例です。

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

ただし、`<Suspense>`境界を多用しchunkの配信を分割しすぎると、[ポップコーンUI](#ポップコーンui)を引き起こす可能性があるため、慎重に設計しましょう。

## トレードオフ

### 下層Routeと共有される`layout.tsx`

レイアウトはRouteにおけるStatic Shell部分とみなすことが可能ですが、Next.jsのレイアウトは[Nesting Layouts↗︎](https://nextjs.org/docs/app/getting-started/layouts-and-pages#nesting-layouts)なため、下層Routeと共有されることに注意が必要です。

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
│     ├── layout.tsx // `/products/[id]`固有のLayout
│     └── page.tsx
└── reviews/
    ├── layout.tsx // `/products/[id]/reviews`固有のLayout
    └── page.tsx
```

#### 明示的な`<Suspense>`境界

レイアウトは共有レイアウトのみを記述する場所と割り切り、ページ内に`<Suspense>`境界を明示的に追加することも戦略として考えられます。

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

[ポップコーンUI↗︎](https://zenn.dev/akfm/articles/popcorn-ui-anti-pattern)はUIが五月雨的に切り替わることを指し、ポップコーンが弾ける様子を揶揄したものです。短時間に画面がチラつくような体験は、ユーザーにとって不快な体験となります。

末端近くのコンポーネントでデータフェッチごとに`<Suspense>`を乱用すると、ポップコーンUIを引き起こす可能性があります。ユーザーにとって意味のあるUIのまとまり単位やボトルネックとなるコンテンツ単位で`<Suspense>`境界を設計することを意識しましょう。

### Cumulative Layout Shift

**Cumulative Layout Shift**（CLS）は[Core Web Vitals↗︎](https://web.dev/articles/vitals?hl=ja#core-web-vitals)の一つで、ページ読み込み中に発生する予期せぬレイアウトの変化を数値化した指標です。Static Shellにフォールバックが含まれるということは、読み込み中にレイアウトが変化する可能性とトレードオフになることでもあります。

`<Suspense>`境界はLayout Shiftを意識して、スケルトンUIの活用や配信順を適切に設計しましょう。
