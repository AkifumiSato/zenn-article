---
title: "Partial Pre Rendering(PPR)"
---

## 要約

PPRは従来のレンダリングモデルのメリットを組み合わせて、シンプルに整理した新しいアプローチです。外側をstatic rendering、`<Suspense>`境界内をdynamic renderingとすることが可能で、既存のモデルを簡素化しつつも高いパフォーマンスを実現します。PPRの使い方・考え方・実装状況を理解しておきましょう。

:::message alert
本稿はNext.js v15.0.0-rc.0時点の情報を元に執筆しており、PPRはさらにexperimentalな機能です。v15.0.0のリリース時や、PPRがstableな機能として提供される際には機能の一部が変更されてる可能性がありますので、ご注意下さい。
:::

:::message
本章は筆者の過去の記事の内容とまとめになります。より詳細にPPRについて知りたい方は以下をご参照ください。
https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering
:::

## 背景

従来Next.jsは[SSR](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)をサポートしてきました。App Routerではこれらに加え、[Streaming SSR](https://nextjs.org/docs/app/building-your-application/rendering/server-components#streaming)もサポートしています。従来Next.jsではレンダリングモデルやそれに付随するオプションが多数あり、そのために複雑化している・考えることが多すぎると多くの開発者が感じていました。

App Routerはこれらをできるだけシンプルに整理するために、サーバー側でのレンダリングをstatic renderingとdynamic renderingという2つのモデルに再整理しました。

| レンダリング                                                                                                                   | タイミング            | Pages Routerとの比較 |
| ------------------------------------------------------------------------------------------------------------------------------ | --------------------- | -------------------- |
| [static rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default) | build時やrevalidate後 | SSG・ISR相当         |
| [dynamic rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)       | ユーザーリクエスト時  | SSR相当              |

しかし、v14までこれらはページやレイアウト単位でしか選択することができなかったため、ログイン機能などを伴うようなパーソナライズされたコンテンツとパフォーマンスはトレードオフの関係にありました。

## 設計・プラクティス

[Partial Pre Rendering(PPR)](https://nextjs.org/docs/app/api-reference/next-config-js/partial-prerendering)はこれらをさらに整理し、ページはstatic rendering、`<Suspense>`境界内をdynamic renderingとすることが可能となります。これにより必ずしもレンダリングをページやレイアウト単位で考える必要はなくなり、「静的な部分をまず返して」「遅れて動的な部分を返す」というシンプルで高いパフォーマンスを得られるメンタルモデルでNext.jsを扱うことができるようになります。

以下は[公式チュートリアル](https://nextjs.org/learn/dashboard-app/partial-prerendering)からの引用画像です。

![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

レイアウトや商品情報についてはstatic renderingで構成されていますが、カートやレコメンドといったユーザーごとに異なるであろう部分はdynamic renderingとすることが表現されています。

### PPRの使い方

### PPRの状況

## トレードオフ

TBW
