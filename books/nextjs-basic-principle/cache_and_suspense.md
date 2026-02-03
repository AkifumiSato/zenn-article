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

Dynamic Contentはネットワーク通信などを伴う動的な処理や、[Runtime Data](https://nextjs.org/docs/app/getting-started/cache-components#runtime-data)、[非決定的操作](https://nextjs.org/docs/app/getting-started/cache-components#non-deterministic-operations)を扱うことができる、`<Suspense>`境界内のコンポーネント群を指します。

:::message
ただし、Pre-Rendering時に解決されうるような動的処理はStatic Shell内でも扱うことが可能です。具体的には以下などが挙げられます。

- `fs.readFileSync(path)`
- `await import(module)`
- `JSON.parse(content)`

:::

### Cached Data(Component)

`"use cache"`が宣言された関数やコンポーネントはStatic Shellに含まれる以外にも、Dynamic Content内から参照することが可能です。この場合`"use cache"`は、デフォルトではインメモリCacheのマーカーとして機能します。

:::message
Vercelでは意図して、Dynamic Content内における`"use cache"`をCacheしないことがあるようです。詳細な仕様や背景については以下issueの解説をご参照ください。

https://github.com/vercel/next.js/issues/85240#issuecomment-3560124078
:::

## 設計・プラクティス

Cache ComponentsでDynamic Contentを扱うには、`<Suspense>`境界が必要です。Routeに`loading.tsx`が存在する場合、Next.jsはPageに対し`<Suspense>`を設定するので、Cache Componentsでは積極的に`loading.tsx`を活用しましょう。

### `loading.tsx`の活用

TBW

- `loading.tsx`が設定されている場合、Next.jsは`page.tsx`に対し`<Suspense>`境界を追加する
  - これにより、`page.tsx`はDynamic Renderingとなる
  - `layout.tsx`はStatic Rendering、`page.tsx`はDynamic Renderingのように分けることができる
    - `layout.tsx`は下層Routeと共有されるので注意（トレードオフへ）

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

### `page.tsx`や`layout.tsx`の一部のみが動的なRoute

- `page.tsx`や`layout.tsx`のごく一部のみが動的な場合は、`loading.tsx`ではなく明示的に`<Suspense>`境界を追加する方が良いこともある
  - e.g. ブログ記事でコメント部分のみ動的な場合を例に解説
  - Cache ComponentsはPPRとなっているため、`layout.tsx`自体をDynamic RenderingにしてしまうとPPRのメリットが薄れてしまう。なのでできるだけ静的にすべき

### パフォーマンスチューニング

- `page.tsx`の一部が低速な場合には、明示的な`<Suspense>`境界を追加することでクライアントへの送信を分割することで、パフォーマンスが改善する可能性がある
  - （実装例追加）
  - 乱用するとポップコーンUIになる（リンク追加）

## トレードオフ

### 下層Routeと共有される`layout.tsx`

- 前述の通りCache Componentsにおいて`layout.tsx`はRouteの静的部分の記述先として使うことができる
- しかし、`layout.tsx`は下層Routeと共有されるため、扱いづらいことも多い
- `layout.tsx`が下層Routeに共有されるのを避けたい場合にはいくつか手段がある
  - 推奨: Route Groupを使って共有レイアウトの`layout.tsx`とRoute固有レイアウトの`layout.tsx`を分ける
  - layout.tsxは共有レイアウトとし、Route固有部分 (`page.tsx`) は全て動的にする
  - layout.tsxは共有レイアウトとし、`loading.tsx`は使わず、`page.tsx`内に`<Suspense>`を記述する

```
(products)/[id]/
├── _lib/
│   └── api/
│       ├── layout.tsx
│       └── product-fetcher.ts
├── (index)/
│     ├── layout.tsx
│     └── page.tsx
└── reviews/
    ├── layout.tsx
    └── page.tsx
```

### ポップコーンUI

- ポップコーンUIの定義
- `<Suspense>`の乱用に気をつけよう

### Cumulative Layout Shift（CLS）

- `fallback`が表示されるということは、CLSが発生するということ
  - スケルトンUIの活用
  - ページ下部を先に表示しない
