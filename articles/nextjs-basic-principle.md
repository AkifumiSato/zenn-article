---
title: "Next.jsにおける設計思想と基本パターン"
emoji: "📕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

Next.jsにおける設計哲学は、Pages RouterとApp Routerで大きく異なります。Next.jsはApp Routerへの移行を推奨していますが、プロダクションでApp Routerを採用してる事例は日本ではまだまだ少ないため、設計思想や基本パターンが普及してないように感じます。

本稿は、筆者なりにNext.jsにおける設計思想と基本パターンをまとめたもので、大きく3つのセクションに分けてこれらを解説します。

- [Server Componentsとfetch](#Server-Componentsとfetch)
  - [1. データ取得はServer Components、データ操作はServer Actions](#1-データ取得はServer-Componentsデータ操作はServer-Actions)
  - [2. 必要なデータを必要な場所で](#2-必要なデータを必要な場所で)
  - [3. Parallel Data Fetchingを意識する](#3-Parallel-Data-Fetchingを意識する)
- [Server ComponentsとRendering](#Server-ComponentsとRendering)
  - [4. static/dynamic renderingを意識する](#4-staticdynamic-renderingを意識する)
  - [5. `<Suspense>`/Streamingを制す](#5-SuspenseStreamingを制す)
  - [6. Router Cacheに注意する](#6-Router-Cacheに注意する)
- [Client Components](#Client-Components)
  - [7. Compositionパターンを駆使し、不用意にClient Componentsを増やさない](#7-Compositionパターンを駆使し不用意にClient-Componentsを増やさない)
  - [8. Presentational/Containerパターンを意識する](#8-PresentationalContainerパターンを意識する)

まだApp Routerに不慣れで勘所が掴めないという方の参考になれば幸いです。

:::message
App Routerの基本的な機能や用語については前提知識としており、本稿では解説しないのでご注意ください。
:::

## Server Componentsとfetch

React Server Componentsにおいて、データ取得・操作をどう考えるかは最も重要なポイントの1つです。これを誤ってしまうと、アーキテクチャのメリットを教授できない可能性があります。

:::message
本稿では**React Server Components**の定義を「Server ComponentsとClient ComponentsからなるReactの新しいアーキテクチャ」として扱います。React Server Components＝Server Componentsではないのでご注意ください。
:::

### 1. データ取得はServer Components、データ操作はServer Actions

React Server Componentsでは、**データ取得はServer Components・データ操作はServer Actionsを利用することが推奨**されています。サーバー側なら高速で安全、そしてシンプルな実装が可能なためです。

これはNext.jsの[Patterns and Best Practices](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#fetching-data-on-the-server)でも示されています。「可能な限り」と書いてはありますが、筆者が考える限りのUIパターンにおいてこれらの設計が成り立たない=クライアント側でデータ取得や操作を行わないといけない事情というのは、ほとんどないように思われます。

データ取得・データ操作の処理は極力サーバー側に実装することを心がけましょう。

### 2. 必要なデータを必要な場所で

従来Pages Routerでは[getServerSideProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)/[getStaticProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-static-props)でまずデータ取得を行い、ページにpropsで渡すという構成がとられていました。これは実際に利用する末端のコンポーネントまでデータの**バケツリレー**を生み出し、冗長な実装の原因となりえました。一方App Routerにおいては、Server Componentsにより**データを参照する末端のコンポーネントでデータ取得を行うことができる**ようになりました。

しかし、末端のコンポーネントでデータ取得を実装するとN+1 fetchを生み出すのではないかと懸念される方もいらっしゃると思います。App RouterではこのようなN+1 fetchを避ける手段として、[Request Memoization](https://nextjs.org/docs/app/building-your-application/caching#request-memoization)が実装されています。

```ts
async function getItem() {
  const res = await fetch('https://.../item/1')
  return res.json()
}
 
const item = await getItem() // cache MISS

// ...

const item = await getItem() // cache HIT
```

Request Memoizationはメモ化なので、当然ながら`fetch()`の引数に同じURLとオプション指定が必要です。複数のコンポーネントで実行しうるデータ取得は関数として抽出(上記例における`getItem()`)しておくことが望ましいでしょう。

### 3. Parallel Data Fetchingを意識する

依存関係のない複数のデータ取得を行う場合、データ取得がSequential(直列)になってしまうとデータ取得が増えるだけパフォーマンスが劣化していきます。以下は[公式ドキュメント](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#parallel-and-sequential-data-fetching)にあるイメージです。

![データ取得のウォーターフォール](/images/nextjs-basic-principle/sequential-fetching.png)

当然のことながら、Parallel(並列)にデータ取得してる方が効率的です。

具体的には`fetch`で作成されるPromiseを`Promise.all()`に渡せば、データ取得のPromiseを並列で実行することが可能です。

```tsx
// ❌ `artist`の取得が終わらないと`albums`の取得が始まらない
const artist = await getArtist(username)
const albums = await getArtistAlbums(username)
```

```tsx
// ✅ `artist`と`albums`の取得が並列で行われる
const artistData = getArtist(username)
const albumsData = getArtistAlbums(username)
const [artist, albums] = await Promise.all([artistData, albumsData])
```

## Server ComponentsとRendering

### 4. static/dynamic renderingを意識する

### 5. `<Suspense>`/Streamingを制す

### 6. Router Cacheに注意する

## Client Components

### 7. Compositionパターンを駆使し、不用意にClient Componentsを増やさない

### 8. Presentational/Containerパターンを意識する

https://quramy.medium.com/react-server-component-%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8-container-presentation-separation-7da455d66576

## その他参考

- [一言で理解するReact Server Components
  ](https://zenn.dev/uhyo/articles/react-server-components-multi-stage)
