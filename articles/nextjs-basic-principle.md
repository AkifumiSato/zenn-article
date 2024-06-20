---
title: "Next.jsにおける設計思想と基本パターン"
emoji: "📕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

Next.jsにおける設計哲学は、Pages RouterとApp Routerで大きく異なります。Next.jsはApp Routerへの移行を推奨していますが、プロダクションでApp Routerを採用してる事例は日本ではまだまだ少ないため、設計思想や基本パターンが普及してないように感じます。

そこで本稿では、筆者なりにNext.jsにおける設計思想と基本パターンをまとめてみました。本稿では大きく3つのセクションに分けてこれらを解説します。

- [Server Componentsとfetch](#Server-Componentsとfetch)
- [Server ComponentsとRendering](#Server-ComponentsとRendering)
- [Client Components](#Client-Components)

本稿がまだApp Routerに不慣れで勘所が掴めない、という方の参考になれば幸いです。

## Server Componentsとfetch

React Server Componentsアーキテクチャにおいて、データ取得・操作をどう考えるかは最も重要なポイントの1つです。これを誤ってしまうと、アーキテクチャのメリットを教授できない可能性があります。

### データ取得はServer Components、データ操作はServer Actions

React Server Componentsアーキテクチャにおいては、データ取得は可能な限りServer側で行うこと、そしてデータ操作に関してはServer Actionsを利用することが推奨されています。Server側なら高速で安全、そしてシンプルな実装が可能なためです。これらは以下で言及されています。

https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#fetching-data-on-the-server

公式にも「可能な限り」と書いてはありますが、筆者が考える限りのUIパターンにおいてこれらの設計が成り立たない=Client側でデータ取得や操作を行わないといけない事情というのは、ほとんどないように思われます。Client側でのデータ取得・操作はアンチパターンだと思ってServer側で実装するように心がけましょう。

### 必要なデータを必要な場所で

従来のPages Routerのアーキテクチャではまずデータ取得を行い、ページにpropsで渡すという構成がとられていました。これはデータの**バケツリレー**を生み出し冗長な実装の原因となりえました。一方App Routerにおいては、Server Componentsによりデータを参照するComponentの近くでデータ取得の実装を行うことができるようになりました。

しかし、末端のComponentでデータ取得を実装するとN+1 fetchを生み出すのではないかと懸念される方もいらっしゃると思います。App RouterではこのようなN+1 fetchを避ける手段として、[Request Memoization](https://nextjs.org/docs/app/building-your-application/caching#request-memoization)が実装されています。この機能を活かすためには、`fetch`に同じURLとオプションを指定することが必要です。容易にMemoizationを活かすためにも、多くの場合データアクセス層を分離しておくことが望ましいでしょう。

```ts
// ❌
const data = await fetch(DUMMY_API_URL, {
  method: "GET",
  headers: {
    "ORIGINAL_HEADER": "hoge", // fetchする際にここが一致してないとキャッシュされない！
  },
}).then((res) => res.json())

// ✅
export async function dummyApiFetchData() {
  return fetch(DUMMY_API_URL, {
    method: "GET",
    headers: {
      "ORIGINAL_HEADER": "hoge", // データアクセス層があれば1回で済むし指定漏れのミスも防げる
    },
  }).then((res) => res.json())
}

const data = await dummyApiFetchData()
```

### Parallel Data Fetchingを意識する

## Server ComponentsとRendering

### static/dynamic renderingを意識する

### `<Suspense>`/Streamingを制す

### Router Cacheに注意する

## Client Components

### Compositionパターンを駆使し、不用意にClient Componentsを増やさない

### Presentational/Containerパターンを意識する

https://quramy.medium.com/react-server-component-%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8-container-presentation-separation-7da455d66576

## その他参考

- [一言で理解するReact Server Components
  ](https://zenn.dev/uhyo/articles/react-server-components-multi-stage)
