---
title: "はじめに"
---

Next.jsの[App Router](https://nextjs.org/docs/app)は、[React Server Components](https://ja.react.dev/learn/creating-a-react-app#which-features-make-up-the-react-teams-full-stack-architecture-vision)をサポートしている先進的なフレームワークであり、Pages Router^[Next.jsに従来から存在するRouter]とは機能・設計・プラクティスなどあらゆる面で大きく異なります。

本書は、App RouterやReact Server Componentsの根底にある**考え方**に基づいた設計やプラクティスをまとめたものです。公式ドキュメントはじめ、筆者や筆者の周りで共有されている理解・前提知識などを元にまとめています。

本書を通じて、Next.jsに対する読者の理解を一層深めることができれば幸いです。

:::message alert
本書は執筆時の最新であるNext.js v15系を前提としています。
:::

## 対象読者

対象読者は以下を想定してます。

- App Routerを説明できるが深く使ったことはない初学者
- App Routerを用いた開発で苦戦している中級者

初学者にもわかりやすい説明になるよう心がけましたが、本書は**入門書ではありません**。そのため、前提知識として説明を省略している部分もあります。入門書としては[公式のLearn](https://nextjs.org/learn)や[実践Next.js](https://gihyo.jp/book/2024/978-4-297-14061-8)などをお勧めします。

https://nextjs.org/learn

https://gihyo.jp/book/2024/978-4-297-14061-8

## 変更履歴

- [2025/01](https://github.com/AkifumiSato/zenn-article/pull/69/files): Next.js v15対応、[[Experimental] Dynamic IO](part_3_dynamicio)追加
- [2024/10](https://github.com/AkifumiSato/zenn-article/pull/67/files): [Server Componentsの純粋性](part_4_pure_server_components)や[第5部 その他のプラクティス](part_5)の追加、一部校正
- [2024/08](https://github.com/AkifumiSato/zenn-article/pull/65/files): 初稿
