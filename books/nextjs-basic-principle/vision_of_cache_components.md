---
title: "Cache Componentsの世界観"
---

## 要約

## 背景

- 従来App Routerにおいて、Cacheは開発者の混乱の元だった
- この2年ほどで、Next.js開発チームはCacheの改善を着実に行なってきたが、それでもなおCacheは複雑だった
- 複雑に感じる最も大きな原因は、多層のCacheがOpt-outで設計されていたことによるもの
- Next.js開発チーム

## 設計・プラクティス

- Cache Components: 再設計されたCacheの世界観
- PPR・DynamicIO・`"use cache"`が統合された
- 開発者は非同期IOを扱うのに、Suspenseか`"use cache"`を明示的に選択する必要がある
- これにより、ユーザーに即座にフィードバックするような実装を促しつつ、Cacheするしないを開発者が明示的に選択することができる

### Suspense: リクエスト時レンダリング

TBW

### `"use cache"`: プリレンダリング

TBW

### Activityによる状態の復元

TBW

## トレードオフ

### ポップコーンUI

TBW

### Activity≠BF Cache的挙動

TBW
