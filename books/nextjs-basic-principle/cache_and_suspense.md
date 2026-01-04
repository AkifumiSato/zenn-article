---
title: "Dynamic Renderingの境界"
---

## 要約

リクエスト時までレンダリング遅延するには`loading.tsx`による暗黙的な`<Suspense>`境界追加か、明示的な`<Suspense>`境界が必要です。ポップコーンUIにならないような`<Suspense>`境界を見極めましょう。

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

### 非同期だがStatic Shellに含まれる処理

TBW

### ポップコーンUI

TBW
