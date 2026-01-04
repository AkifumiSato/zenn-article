---
title: "Cache Componentsの世界観"
---

## 要約

Cache Componentsは、PPRとCacheヒューリスティックの改善により、**デフォルトで高いパフォーマンス**と**優れた開発者体験**の両立を目指したものです。

開発者はデータフェッチなどの動的な処理を扱う際に、`"use cache"`によるStatic Shellへの追加か`<Suspense>`によるリクエスト時までレンダリング遅延するか、どちらかを選択する必要があります。

## 背景

- Next.jsのコンセプト
  - 「デフォルトで高いパフォーマンス」と「優れた開発者体験」
  - App Routerではこれらに基づいて、CacheはOpt-out可能だができるだけ開発者が気にしなくていいように設計されていた
- 開発者の混乱
  - しかし現実には、Cacheは開発者の混乱の元だった
  - WebアプリケーションにおいてCacheできないものを扱うことは一般的で、Opt-outするにはCacheを気にするしかなかった
- 着実な改善と根本的な課題
  - Next.js開発チームは少しづつ慎重にCacheの改善を重ね、v13~15で確実に良くなった
  - しかし、それでもなおCacheは複雑だった
  - 根本的な複雑さは、Opt-out型の設計に起因してると考えられた
- Next.js開発チームのジレンマ
  - Opt-in型にすれば「デフォルトで高いパフォーマンス」が犠牲になる
  - 何も変えなければ「優れた開発者体験」が犠牲になり続ける
  - この非常に困難なジレンマを解いたのが**Cache Components**

## 設計・プラクティス

- Cache Components
  - 再設計されたCacheの世界観
  - PPR・DynamicIO・`"use cache"`が統合された

### PPR(Partial Pre-Rendering)

TBW

### Dynamic IO

TBW

### `"use cache"`

TBW

### その他: Activity

TBW

## トレードオフ

### Activity≠BF Cache

TBW
