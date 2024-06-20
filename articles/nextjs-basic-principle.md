---
title: "Next.jsにおける設計思想と基本パターン"
emoji: "📕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

Next.jsにおける設計哲学は、Pages RouterとApp Routerで大きく異なります。Next.jsはApp Routerへの移行を推奨していますが、プロダクションでApp Routerを採用してる事例は日本ではまだまだ少ないため、設計思想や基本パターンが普及してないように感じます。

そこで本稿では、筆者なりにNext.jsにおける設計思想と基本パターンをまとめてみました。まだApp Routerに不慣れで勘所が掴めないという方の参考になれば幸いです。

## Server Componentsとfetch

### データ取得はServer Components、データ操作はServer Actions

### 必要なデータを必要な場所で

### データアクセス層を設けてmemoizationを意識する

### Parallel Data Fetchingを意識する

## Server ComponentsとRendering

### static/dynamic renderingを意識する

### `<Suspense>`/Streamingを制す

## Client Components

### Compositionパターンを駆使し、不用意にClient Componentsを増やさない

### Router Cacheに注意する

### Presentational/Containerパターンを意識する

https://quramy.medium.com/react-server-component-%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8-container-presentation-separation-7da455d66576

## その他参考

- [一言で理解するReact Server Components
  ](https://zenn.dev/uhyo/articles/react-server-components-multi-stage)
