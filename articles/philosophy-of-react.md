---
title: "Reactと現実世界をつなぐescape hatches、そしてReact Server Component"
emoji: "⚛️"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

最近、[react.dev](https://react.dev/)がリリースされました。この新しい公式サイトは、hooks時代に適用した新たなドキュメントサイトであり、入門学習のためのドキュメントとAPIドキュメントを兼ねています。このサイトはかなり力が入ってて、見てるといろんな発見があります。

個人的には中でも、learnにある[Escape Hatches](https://react.dev/learn/escape-hatches)の章に興味が惹かれました。本稿ではescape hatchesやReact Server Componentsを通してReactがデータフェッチをどう考えてきたのかを自分なりにまとめてみます。

## Escape Hatches

Reactは宣言的UIを実現するべく設計されており、複雑化しやすい状態管理やイベントリスナーをうまく抽象化・制限するよう設計されています。一方で、Reactだけで現実世界をすべて表現できるわけではありません。外部システムとの同期やブラウザAPIの利用のように、Reactの外に出て何かしら処理を行う必要がある場合があります。こういった現実のアプリケーション実装のニーズのために、Reactでは`useEffect`や`useRef`などのAPIが用意されており、**escape hatches**と表現されています。

escape hatchesは、Reactの慣用的なパターンで対処できない時に利用されることが想定されています。そのため、[escape hatchesの利用は避けれるなら避けるべきです](https://react.dev/learn/you-might-not-need-an-effect)。

### useEffectでのデータ取得

さて、escape hatchesの一例にuseEffectを上げましたが、クライアントサイドでのデータ取得は非常に一般的に行われる実装です。データ取得の処理はReactの外の世界で行われるものであり、サードパーティに依存せずに行うには、escape hatches（`useEffect`）を利用して、データ取得の結果をReactに渡す必要がありました。

これに関してreact.devでは、[Next.jsなどのフレームワークのデータ取得の利用や、swr・react-queryなどのライブラリを利用することを推奨](https://react.dev/learn/synchronizing-with-effects#what-are-good-alternatives-to-data-fetching-in-effects)しており、実際、このようなフレームワークやライブラリを採用することは非常に多いです。

### Escape Hatchesまとめ

- Reactは、Reactの慣用的なパターンで対処できない時のために、escape hatchesを提供している
- escape hatchesの利用は避けれるなら避けるべき
- データフェッチについてReactでは、[Next.jsなどのフレームワークのデータ取得の利用や、swr/react-queryなどのライブラリを利用することを推奨](https://react.dev/learn/synchronizing-with-effects#what-are-good-alternatives-to-data-fetching-in-effects)している

## React Server Components

クライアントサイドのデータフェッチについて、swr/react-queryなどのライブラリを利用することで車輪の再開発からの解放や一貫した設計・サポートを得られる反面、コンポーネントを強く意識したAPIの設計や実装、もしくはコンポーネントに最適化されていないAPIの利用による複雑化などの課題を抱えやすいことがわかってきました。

これらの課題を解消するのが**React Server Components**（以下RSC）です。RSCのMotivationでは、上記課題の本質は、アプリケーションを構築する上でReactがクライアント中心であるためにサーバーをうまく活用できていないことだった、とされています。

https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md#motivation

### RSCの概要

**Serverコンポーネント**はサーバー側でデータを取得・レンダリングし、その結果をクライアントで描画します。**Clientコンポーネント**は、従来のReactコンポーネント同様にクライアント側でレンダリングすることに重きを置いたコンポーネントです。これは新しいものを意味するものではなく、Serverコンポーネントと区別するために命名されたもので、`use client`ディレクティブを記述する以外に従来のコンポーネントと違いはありません。

https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md#basic-example

> The name “Client Component” doesn’t mean anything new, it only serves to distinguish these components from Server Components.

:::message
Clientコンポーネントはサーバーコンポーネントを直接importすることはできません。これは、クライアント側のchunkにサーバー固有のロジックが含まれるわけにはいかないことからも明らかです。
:::

これは、**従来のクライアント中心のReactの概念を変えるものであり、データ取得をescape hatch依存ではなく、Reactの外の世界に出ずにデータ取得を行うことができるようにする**ものであると言えます。

現在、RSCはNext.jsの[appディレクトリ](https://nextjs.org/docs/advanced-features/custom-app)で採用されていますが、まだ開発中の試験的機能のため、今後のリリースで安定化されていくことが待たれます。

## Presentational/Containerコンポーネント

少し話はそれますが、Reactには**Presentational/Containerコンポーネント**という設計パターンが存在します。これはかつて[Dan abramov](https://twitter.com/dan_abramov)が提唱したhooks登場前のデータと振る舞いを分離するためのパターンです。

:::message
ただし、これはhooks登場前に考えられたものであり、現在はあまり使うべきパターンではない旨が下記の記事の冒頭でも述べられています。
:::

https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0

Presentationalコンポーネントはデータを受け取り、それを描画するだけのコンポーネントです。Containerコンポーネントはグローバルな状態やAPIからデータを取得し、それをPresentationalコンポーネントに渡すコンポーネントです。

Presentational/ContainerコンポーネントもServer/Clientコンポーネントも、どちらもデータ取得を中心に考えてコンポーネントを切り分ける必要があります。もちろん、アーキテクチャ的には全く異なるものなのですが、見様によっては[技術の螺旋](https://speakerdeck.com/twada/understanding-the-spiral-of-technologies?slide=10)が一周したのかもしれません。

https://twitter.com/dan_abramov/status/1639824633124757504

## まとめ

- 現在のReactは、クライアントサイド中心の世界であり、クライアントサイドでのデータ取得はescape hatchesに依存しており、Reactの外に出る必要がある
- これについてreact.devでは、Next.jsやRemixなどのフレームワークや、swr・react-queryなどのライブラリを利用することを推奨している
- React Server Componentsは、サーバー側でデータ取得を行い、その結果をクライアント側で描画することができるようにするものである
- これは、従来のクライアント中心のReactの概念を変えるものである
