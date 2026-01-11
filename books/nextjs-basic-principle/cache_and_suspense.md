---
title: "Dynamic Renderingの境界"
---

## 要約

Cache Componentsの世界観にはPPRが含まれており、Dynamic Renderingを扱うには`loading.tsx`か明示的な`<Suspense>`境界が必要です。`<Suspense>`境界の見極めは、パフォーマンスやUXに直結するので慎重に検討しましょう。

## 背景

- Cache Componentsでは、1つのRouteをStatic Shell・動的コンテンツ・Cacheされたデータやコンポーネントに分解して考えることができる
- 全てのRouteはPre-Renderingされる。リクエスト時までレンダリングを遅延するには`<Suspense>`境界（とDynamic APIsなどへのアクセス）が必要

## 設計・プラクティス

Cache ComponentsにおいてDynamic Renderingを扱うには、`<Suspense>`境界内で扱う必要があります。`loading.tsx`が設定されている場合、Next.jsはPageに対し`<Suspense>`境界を追加します。

### 動的な要素が多いRoute

- 動的な要素が多いRouteでは、`loading.tsx`による`<Suspense>`境界の追加を活用しましょう
  - これにより、`page.tsx`はDynamic Renderingとなる
    - memo: 厳密には`page.tsx`や`layout.tsx`からdefault exportされてるコンポーネント、説明のわかりやすさからファイル名で説明すること明記
  - 静的なレイアウト部分は`layout.tsx`で記述
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

- `page.tsx`や`layout.tsx`のごく一部のみが動的な場合、明示的に`<Suspense>`境界を追加する
  - e.g. 商品ページなど、実装例を追記

### パフォーマンスチューニング

- 動的な要素が多いRouteかつ、一部のレンダリングをパフォーマンス観点で遅延させたい場合には明示的に`<Suspense>`境界を追加することが有効
- 乱用するとポップコーンUIになる（リンク追加）

## トレードオフ

### 下層Routeと共有される`layout.tsx`

- `layout.tsx`は下層Routeと共有されるため、扱いづらいことも多い
- `layout.tsx`が下層Routeに共有されるのを避けたい場合
  - Route Groupを使って共有レイアウトの`layout.tsx`とRoute固有レイアウトの`layout.tsx`を分けることを推奨

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

- 他にも以下の手段がある
  - layout.tsxは共有レイアウトとし、Route固有部分 (`page.tsx`) は全て動的にする
  - layout.tsxは共有レイアウトとし、`loading.tsx`は使わず、`page.tsx`内に<Suspense>を記述する

### ポップコーンUI

- ポップコーンUIの定義
- `<Suspense>`の乱用に気をつけよう

### Cumulative Layout Shift（CLS）

- `fallback`が表示されるということは、CLSが発生するということ
  - スケルトンUIの活用
  - ページ下部を先に表示しない
