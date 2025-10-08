---
title: "はじめに"
---

Next.jsには2つのRouterが同梱されています。

- [App Router↗︎](https://nextjs.org/docs/app): 現在主流の新しいRouter
- [Pages Router↗︎](https://nextjs.org/docs/pages): 従来のRouter

App Routerは[React Server Components↗︎](https://ja.react.dev/learn/creating-a-react-app#which-features-make-up-the-react-teams-full-stack-architecture-vision)（**RSC**）はじめReactの先進的な機能をサポートするフレームワークであり、Pages Routerとは機能・設計・プラクティスなどあらゆる面で大きく異なります。

本書は、Next.jsやRSCの根底にある**考え方**に基づいた設計やプラクティスをまとめたものです。公式ドキュメントはじめ、筆者や筆者の周りで共有されている理解・前提知識などを元にまとめています。

本書を通じて、Next.jsに対する読者の理解を一層深めることができれば幸いです。

:::message alert

- 本書はNext.js v15系を前提としています。
- 本書では断りがない限り「Next.js」はApp Routerを指します。

:::

## 対象読者

対象読者は以下を想定してます。

- Next.jsを説明できるが深く使ったことはない初学者
- Next.jsを用いた開発で苦戦している中級者

初学者にもわかりやすい説明になるよう心がけましたが、本書は**入門書ではありません**。そのため、前提知識として説明を省略している部分もあります。入門書としては[公式のLearn↗︎](https://nextjs.org/learn)や[実践Next.js↗︎](https://gihyo.jp/book/2024/978-4-297-14061-8)などをお勧めします。

https://nextjs.org/learn

https://gihyo.jp/book/2024/978-4-297-14061-8

## 謝辞

本書の執筆にあたり、[koichikさん↗︎](https://x.com/koichik)にレビュー協力をいただきました。多大な時間を割いてレビューや議論にお付き合いいただいたおかげで、本書をより良いものにできました。本当にありがとうございます。

## 変更履歴

- 2025/10: [クライアントとサーバーのバンドル境界](part_2_bundle_boundary)追加、全体の改定<!-- https://github.com/AkifumiSato/zenn-article/pull/81 -->
- 2025/01: Next.js v15対応<!-- https://github.com/AkifumiSato/zenn-article/pull/69 -->
- 2024/10: [Server Componentsの純粋性](part_4_pure_server_components)や[第5部 その他のプラクティス](part_5)の追加、一部校正<!-- https://github.com/AkifumiSato/zenn-article/pull/67 -->
- 2024/08: 初稿<!-- https://github.com/AkifumiSato/zenn-article/pull/65 -->
