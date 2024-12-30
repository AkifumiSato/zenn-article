---
title: "[Experimental] Dynamic IO"
---

## 要約

Next.jsは現在、キャッシュの仕組みの大幅な刷新に取り組んでいます。これにより、本書で紹介してきたキャッシュに関する知識の多くは**過去のものとなる可能性**があります。

これからのNext.jsでキャッシュがどう変わるのか理解して、将来の変更に備えましょう。

:::message alert
本章で紹介するDynamic IOは執筆時現在は実験的機能で、利用するには`canary`バージョンが必要となります。
:::

## 背景

[_第3部_](./part_3)ではApp Routerにおけるキャッシュの理解が重要であるとし、ここまで解説してきましたが、多層のキャッシュや複数の概念が登場し、難しいと感じた方も多かったのではないでしょうか。実際、App Router登場当初から現在に至るまでキャッシュに関する批判的意見は多く、現在のNext.jsにおける最も大きなネガティブ要素と言っても過言ではありません。

https://github.com/vercel/next.js/discussions/54075

一方Next.jsは、**デフォルトで高いパフォーマンスを実現できることを重視したフレームワーク**です。「開発者のフィードバックを重視しキャッシュは全てオプトイン」にすればいいかというと、Next.jsコアチームにとってはコンセプトの1つを蔑ろにする破壊的変更になってしまうため、キャッシュに関する批判への対応は非常に難しい問題でした。

### PPRから見出した解決への道筋

Next.jsコアチームはこの問題に対し、**Partial Pre Rendering**(PPR)によって解決の道筋を見出しました。ページにDynamic RenderingとStatic Renderingを混在可能にすることで、Data Cacheのデフォルトを変更しても至る所に`cache: "force-cache"`を指定しなくても良い、よりシンプルな設計と高いパフォーマンスを両立させることが可能となりました。

![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

本章で紹介する**Dynamic IO**の世界観は、これをさらに発展させシンプルな設計と高いパフォーマンスを両立させることを目指したものです。

:::message
PPRについて詳細を知りたい方は後述の[_Partial Pre Rendering(PPR)_](./part_4_partial_pre_rendering)をご参照ください。
:::

## 設計・プラクティス

**Dynamic IO**は文字通り、Next.jsにおける動的I/O処理の振る舞いを大きく変更するものです。

https://nextjs.org/docs/app/api-reference/config/next-config-js/dynamicIO

具体的には、動的I/O処理とは以下を指します。

- データフェッチ: `fetch()`やDBアクセスなど
- [Dynamic APIs](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-apis): `headers()`や`cookies()`など
- Next.jsがラップするモジュール: `Date`、`Math`、Node.jsの`crypto`モジュールなど
- 任意の非同期関数(マイクロタスクを除く)

Dynamic IOでは、これらの処理を含む場合には`"use cache"`でキャッシュするか`<Suspense>`で遅延するか選択する必要があり、デフォルトではこれらを扱うことができません。

これはデフォルトキャッシュがもたらした混乱に対し、「デフォルトでキャッシュするかしないか」ではなく、「**明示的な選択を強制する**」というアプローチによって、高いパフォーマンスを実現しやすい形をとりつつも開発者の混乱を解消することを目指したもので、筆者はシンプルかつ柔軟な設計だと評価しています。このようなアプローチが可能となったのは、`"use cache"`によるキャッシュ境界を指定可能にする発明が大きく寄与しています。

### `"use cache"`によるキャッシュ

TBW

### `<Suspense>`境界内の遅延実行

TBW

## トレードオフ

### キャッシュの永続化

### キャッシュに関する制約

### `fallback`のレンダリングが必至
