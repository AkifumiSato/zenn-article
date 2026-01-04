---
title: "Dynamic Renderingの境界"
---

## 要約

Cache Componentsの世界観にはPPRが含まれており、Dynamic Renderingを扱うには`loading.tsx`か明示的な`<Suspense>`境界が必要です。これらを設定する際には、ポップコーンUIにならないように注意しましょう。

## 背景

- Cache Componentsでは、1つのRouteをStatic Shell・動的コンテンツ・Cacheされたデータやコンポーネントに分解して考えることができる
- デフォルトでPageはPre-Renderingされるため、`"use cache"`によるStatic Shellへの追加か`<Suspense>`によるリクエスト時までレンダリング遅延するか選ばなければならない

## 設計・プラクティス

- 動的なページでは`loading.tsx`による`<Suspense>`境界追加を活用しよう
- 部分的に遅延レンダリングさせたいなら`<Suspense>`を明示的に宣言しよう

### `loading.tsx`による`<Suspense>`境界の追加

TBW

### 明示的な`<Suspense>`境界の追加

TBW

## トレードオフ

### Prerenderに含まれる動的処理

- `await import()`
- `fs.readFileSync()`

TBW

### ポップコーンUI

TBW
