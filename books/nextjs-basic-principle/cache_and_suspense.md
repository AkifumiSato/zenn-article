---
title: "Dynamic Renderingの境界"
---

## 要約

Cache Componentsの世界観にはPPRが含まれており、Dynamic Renderingを扱うには`loading.tsx`か明示的な`<Suspense>`境界が必要です。これらを設定する際には、ポップコーンUIにならないように注意しましょう。

## 背景

- Cache Componentsでは、1つのRouteをStatic Shell・動的コンテンツ・Cacheされたデータやコンポーネントに分解して考えることができる
- デフォルトでPageはPre-Renderingされるため、Dynamic Renderingを扱うには`<Suspense>`でリクエスト時までレンダリング遅延する境界を宣言する必要があります

## 設計・プラクティス

動的なページにおいて`<Suspense>`境界の設計にはいくつかのパターンがあります。

- `loading.tsx`を配置（大抵はこれで十分）
- ページ全体がDynamic Renderingな場合
- `<Suspense>`
- ページの一部のみがDynamic Renderingな場合
- ページ全体がDynamic Rendering&低速な部分を遅延させるために

### `loading.tsx`による`<Suspense>`境界の追加

- ページ全体がDynamic Renderingな場合
- 大抵はこれで十分

```tsx
// <Layout>: layout.tsx
// <Page>: page.tsx
<Layout>
  <Suspense fallback={<loading.tsx />}>
    <Page />
  </Suspense>
</Layout>
```

TBW

### 明示的な`<Suspense>`境界の追加

- ページの大部分がPre-Rendering可能で一部がDynamic Renderingな場合
- ページ全体がDynamic Renderingだが部分的に低速でStreaming配信を分割したい時
- ポップコーンUIに軽く言及＆リンク

TBW

## トレードオフ

### Prerenderに含まれる動的処理

- `await import()`
- `fs.readFileSync()`

TBW

### `<Suspense>`の乱用とポップコーンUI

- ポップコーンUIの定義
- `<Suspense>`の乱用に気をつけよう

TBW
