---
title: "Reactと現実世界をつなぐescape hatches、そしてReact Server Components"
emoji: "⚛️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

最近、[react.dev](https://react.dev/)がリリースされました。この新しい公式サイトは、hooks時代に適用した新たなドキュメントサイトであり、入門学習のためのドキュメントとAPIドキュメントを兼ねています。このサイトはかなり力が入ってて、見てるといろんな発見があります。

個人的には中でも、learnにある[Escape Hatches](https://react.dev/learn/escape-hatches)の章に興味が惹かれました。本稿ではescape hatchesやReact Server Componentsを通してReactがデータフェッチをどう考えてきたのか、自分なりにまとめてみます。

## Escape Hatches

Reactは宣言的UIを実現するべく設計されており、複雑化しやすい状態管理やイベントリスナーを抽象化・制限するよう設計されています。一方で、現実のアプリケーション実装においては、外部システムとの同期やブラウザAPIの利用のようにReactの外に出て何かしら処理を行う必要がある場合があります。こういった現実のアプリケーション実装のニーズにも対応すべく、Reactでは`useEffect`や`useRef`などのAPIが用意されており、これらを指して**escape hatches**と表現されています。

このescape hathcesという表現は、だいぶ前からReactコアチームでは使われていたようです。下記はReactコアチームの[Andrew Clark](https://twitter.com/acdlite)の2018年のツイートです。

https://twitter.com/acdlite/status/974437383939743746

上記によるとescape hatchesは、Reactの慣用的なパターンで対処できない時に利用されることが想定されています。そのため、escape hatchesの利用は避けれるなら避けるべきです。react.dev内でも下記のように、不要な`useEffect`は削除すべき旨が記載されています。

https://react.dev/learn/you-might-not-need-an-effect

### `useEffect`でのデータ取得

さて、escape hatchesの一例に`useEffect`を上げましたが、クライアントサイドでのデータ取得は非常に一般的に行われる実装です。データ取得の処理はReactの外の世界で行われるものであり、サードパーティに依存せずに行うには、escape hatches（`useEffect`）を利用して、データ取得の結果をReactに渡す必要がありました。

これに関してreact.devでは、Next.jsなどのフレームワークのデータ取得の利用や、swr/react-queryなどのライブラリを利用することを推奨しており、実際、このようなフレームワークやライブラリを採用することは非常に多いと思われます。

https://react.dev/learn/synchronizing-with-effects#what-are-good-alternatives-to-data-fetching-in-effects

### Escape Hatchesまとめ

- Reactは、Reactの慣用的なパターンで対処できない時のために、escape hatches（`useEffect`や`useRef`など）を提供している
- escape hatchesの利用は避けれるなら避けるべき
- データフェッチについてReactでは、Next.jsなどのフレームワークのデータ取得の利用や、swr/react-queryなどのライブラリを利用することを推奨している

## React Server Components

クライアントサイドのデータフェッチについて、swr/react-queryなどのライブラリを利用することで車輪の再開発からの解放や一貫した設計・サポートを得られる反面、コンポーネントを強く意識したAPIの設計や実装、もしくはコンポーネントに最適化されていないAPIの利用による複雑化などの課題を抱えやすいことがわかってきました。

これらの課題を根本的に解消するのが**React Server Components**（以下RSC）です。RSCのMotivationでは、上記課題の本質は、アプリケーションを構築する上でReactがクライアント中心であるためにサーバーをうまく活用できていないことだった、とされています。

https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md#motivation

> The fundamental challenge was that React apps were client-centric and weren’t taking sufficient advantage of the server. If we could allow developers to easily leverage their server more, we could solve all of these challenges and provide a more powerful approach to building apps, small or large.
Deepl訳: 根本的な課題は、Reactアプリがクライアント中心で、サーバーを十分に活用できていないことでした。もし開発者が簡単にサーバーをもっと活用できるようになれば、これらの課題をすべて解決し、規模の大小にかかわらず、アプリを構築するためのより強力なアプローチを提供することができます。

RSCはデータ取得のしやすさだけでなく、パフォーマンス課題も同時に解消することを目指しています。そしてこれらの課題の根本が、「サーバーをうまく活用できていない」ことだったとしています。

### RSCの概要

**Serverコンポーネント**はサーバー側でデータを取得・レンダリングし、その結果をクライアントで描画します。**Clientコンポーネント**は、従来のReactコンポーネント同様にクライアント側でレンダリングすることに重きを置いたコンポーネントです。これは新しいものを意味するものではなく、Serverコンポーネントと区別するために命名されたもので、`use client`ディレクティブを記述する以外に従来のコンポーネントと違いはありません。

https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md#basic-example

> The name “Client Component” doesn’t mean anything new, it only serves to distinguish these components from Server Components.
Deepl訳: 「Clientコンポーネント」という名称に新しい意味はなく、Serverコンポーネントと区別するためのものでしかありません。

:::message
ClientコンポーネントはServerコンポーネントを直接importすることはできません。これは、クライアント側のchunkにサーバー固有のロジックが含まれるわけにはいかないことからも明らかです。
:::

これは、**従来のクライアント中心のReactの哲学を変えるものであり、データ取得をescape hatchに依存せずに行うことができるようにする**ものであると言えます。

現在RSCは開発中の試験的機能ではありますが、Next.jsの[appディレクトリ](https://nextjs.org/docs/advanced-features/custom-app)で採用されています。appディレクトリ自体も当然ながら試験的機能なので、今後Next.jsやReactのリリースでこれらが安定化されていくことが待たれます。

## Presentational/Containerコンポーネント

少し話はそれますが、Reactには**Presentational/Containerコンポーネント**という設計パターンが存在します。これはかつて[Dan abramov](https://twitter.com/dan_abramov)が提唱したhooks登場前のデータと振る舞いを分離するためのパターンです。

:::message
ただし、これはhooks登場前に考えられたものであり、現在はあまり使うべきパターンではない旨が下記の記事の冒頭でも述べられています。
:::

https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0

Presentationalコンポーネントはデータを受け取り、それを描画するだけのコンポーネントです。Containerコンポーネントはグローバルな状態やAPIからデータを取得し、それをPresentationalコンポーネントに渡すコンポーネントです。

Presentational/ContainerコンポーネントもServer/Clientコンポーネントも、どちらもデータ取得を中心に考えてコンポーネントを切り分ける点では似通っています。Server/Clientコンポーネントはchunkの縮小を目的に切り分けることもあり、データ取得の観点のみで切り分けるわけではないですし、アーキテクチャ的には全く異なるものなのですが、見様によっては[技術の螺旋](https://speakerdeck.com/twada/understanding-the-spiral-of-technologies?slide=10)が一周したのかもしれません。

この辺はDan abramovがまた記事にしてくれるかもしれません。

https://twitter.com/dan_abramov/status/1639824633124757504

## まとめ

- 現在のReactはクライアントサイド中心に設計されており、クライアントサイドでのデータ取得はescape hatchesに依存している
- これについてReactは、Next.jsやRemixなどのフレームワークや、swr・react-queryなどのライブラリを利用することを推奨している
- React Server Componentsによって、escape hatchesに依存せずにデータ取得を行うことができるようになる
- これは、従来のクライアント中心のReactの概念を変えるものである
