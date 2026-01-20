---
title: "Dynamic Renderingの境界"
---

## 要約

Cache Componentsの世界観にはPPRが含まれており、Dynamic Renderingを扱うには`loading.tsx`か明示的な`<Suspense>`境界が必要です。`<Suspense>`境界の見極めは、パフォーマンスやUXに直結するので慎重に検討しましょう。

## 背景

- Cache Componentsでは、1つのRouteをStatic Shell・動的コンテンツ・Cacheされたデータやコンポーネントに分解して考えることができる
- 全てのRouteはPre-Renderingされる。リクエスト時までレンダリングを遅延するには`<Suspense>`境界（とDynamic APIsなどへのアクセス）が必要

## 設計・プラクティス

Cache ComponentsにおいてDynamic Renderingを扱うには、`<Suspense>`が必要です。

### `loading.tsx`の活用

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
