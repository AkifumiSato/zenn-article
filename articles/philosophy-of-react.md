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
- Reactの哲学
  - UXを表現するための主要なAPIはreactによって容易されている
  - Reactが表現する世界（以下React世界）は現実世界をすべて表現できるわけではない
  - この乖離をつなぐescape hatchesが用意されている
- escape hatches
  - React世界と現実世界をつなぐための仕組み
  - useEffectやuseRefなどのhooksがこれにあたる
  - escape hatchesの利用は避けれるなら避けるべき
- useEffectでのデータ取得
  - クライアントサイドでのデータ取得は、あまりに一般的な用途
  - データ取得の処理はReactの外の世界で行われるものであり、サードパーティに依存せずに行うには、escape hatches（useEffect）を利用して、データ取得の結果をReactに渡す必要があった
  - これに関してreact.devでは、Next.jsなどのフレームワークのデータ取得の利用や、swr・react-queryなどのライブラリを利用することを推奨している
  - しかしサードパーティに依存する解決は、パフォーマンス課題やバンドルサイズの肥大化など複数の課題を引き起こしていた
  - 本質的には、アプリケーションを構築する上で、reactがクライアント中心であることに限界がきていたとされている（RFCのモチベーションを参照）
- React Server Components
  - 上記の課題を解消するのがReact Server Components
    - サーバーコンポーネントはサーバー側でデータを取得・レンダリングし、その結果をクライアントで描画する
    - クライアントコンポーネントは、クライアント側でのみレンダリングされ、サーバーコンポーネントからpropsを受け取ることができる
    - これは、従来のクライアント中心のreactの概念を変えるものであり、データ取得をescape hatch依存ではなく、reactの標準機能に基づいた世界でデータ取得を行うことができるようにするものである
- Presentational/Container Componentの螺旋
  - RSCはPresentational/ContainerパターンのContainerに似てる
  - Presentational/Container Componentは、かつてDanが提唱したhooks登場前のデータと振る舞いを分離するためのパターン
    - Presentational Componentは、データ取得を行わず、Container Componentからpropsを受け取る
    - Container Componentは、APIやグローバルStateからデータ取得を行い、Presentational Componentにpropsを渡す
    - Server/Client Componentは、このパターンに似てる部分があるが、一方で全く異なるものでもある
  - t-wadaさんの言葉を借りるなら、これは「技術の螺旋」である
- まとめ
  - 現在のreactは、クライアント中心の世界であり、クライアントサイドでのデータ取得はescape hatchesに依存しており、react世界では、データ取得はできない
  - React Server Componentsは、サーバー側でデータ取得を行い、その結果をクライアント側で描画することができるようにするものである

### 参考リンク

- [RSC rfc](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md#motivation)

