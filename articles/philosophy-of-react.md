---
title: "Reactと現実世界をつなぐescape hatches、そしてReact Server Component"
emoji: "⚛️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

## 構成メモ

- 序文
  - 最近、react.devがリリースされた
    - hooks時代に適用した新たなドキュメントサイト
    - 入門学習のためのドキュメントとAPIドキュメントを兼ねてる
    - かなり力が入ってて、いろんな発見がある
  - 中でも、learnにあるescape hatchesの章が面白い
  - これはReactの哲学を表しており、今回はこれの深堀記事
- escape hatches
  - ReactにはUXを実装するためのAPIが豊富に用意されている
  - 一方で、Reactだけで現実世界をすべて表現できるわけではない
    - 例としては、外部システムとの同期であったり、ブラウザAPIの利用などが挙げられる
  - Reactには、Reactと現実世界をつなぐためにescape hatchesが用意されている
  - useEffectやuseRefなどのhooksがこれにあたる
  - escape hatchesの利用は避けれるなら避けるべき
- useEffectでのデータ取得
  - escape hatchesの一例にuseEffectを上げたが、クライアントサイドでのデータ取得は非常に一般的に行われる実装
  - データ取得の処理はReactの外の世界で行われるものであり、サードパーティに依存せずに行うには、escape hatches（useEffect）を利用して、データ取得の結果をReactに渡す必要があった
  - これに関してreact.devでは、Next.jsなどのフレームワークのデータ取得の利用や、swr・react-queryなどのライブラリを利用することを推奨している
  - しかしサードパーティに依存する解決は、パフォーマンス課題やバンドルサイズの肥大化など複数の課題を引き起こしていた
  - 本質的には、アプリケーションを構築する上で、reactがクライアント中心であることに限界がきていたとされている（RFCのモチベーションを参照）
- React Server Components
  - 上記の課題を解消するのがReact Server Components（以下RSC）
    - サーバーコンポーネントはサーバー側でデータを取得・レンダリングし、その結果をクライアントで描画する
    - クライアントコンポーネントは、クライアント側でのみレンダリングされ、サーバーコンポーネントからpropsを受け取ることができる
    - これは、従来のクライアント中心のReactの概念を変えるものであり、データ取得をescape hatch依存ではなく、Reactの外の世界に出ずにデータ取得を行うことができるようにするものである
- Presentational/Container Componentの螺旋
  - RSCはPresentational/ContainerパターンのContainerに似てる
  - Presentational/Container Componentは、かつてDanが提唱したhooks登場前のデータと振る舞いを分離するためのパターン
    - Presentational Componentは、データ取得を行わず、Container Componentからpropsを受け取る
    - Container Componentは、APIやグローバルStateからデータ取得を行い、Presentational Componentにpropsを渡す
    - Server/Client Componentは、このパターンに似てる部分があるが、一方で全く異なるものでもある
  - t-wadaさんの言葉を借りるなら、これは「技術の螺旋」と言えるかもしれない
- まとめ
  - 現在のReactは、クライアントサイド中心の世界であり、クライアントサイドでのデータ取得はescape hatchesに依存しており、Reactの外に出る必要がある
  - これについてreact.devでは、Next.jsやRemixなどのフレームワークや、swr・react-queryなどのライブラリを利用することを推奨している
  - React Server Componentsは、サーバー側でデータ取得を行い、その結果をクライアント側で描画することができるようにするものである
  - これは、従来のクライアント中心のReactの概念を変えるものである

### 参考リンク

- [RSC rfc](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md#motivation)

