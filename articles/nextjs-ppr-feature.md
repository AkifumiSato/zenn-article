---
title: "PPRはSSRとSSGの正当後継になりうるのか"
emoji: "👑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

**Partial Pre-Rendering**(以降PPR)はNext.js v14.0で発表された、SSRやSSGにならぶ**新たなpre-rendering方式**です。

https://nextjs.org/blog/next-14#partial-prerendering-preview

PPRは本稿執筆時点の2024/05現在も開発中で、v15のRC版にてexperimentalフラグを有効にすることで利用することができます。

https://rc.nextjs.org/docs/app/api-reference/next-config-js/ppr

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    ppr: "incremental", // ppr: boolean | "incremental"
  },
}
 
module.exports = nextConfig

// page.tsx(or layout.tsx)
export const experimental_ppr = true
```

PPRはSSR/SSG/ISRなどに並ぶ新たなpre-rendering方式として個人的にとても注目度の高いトピックなのですが、筆者の観測範囲では話題になってるのはまだ一部で、そこまで盛り上がってないように感じます。

本稿を通じてPPRとは何か、何を解決しようとしているのか、そしてなぜPPRが今アツいのかお伝えしたいと思います。

## pre-renderingを振り返る

PPRの話をする前に、これまでのNext.jsのpre-renderingについて振り返ってみましょう。

これまでのNext.jsにおけるpre-renderingは大きく3つありました。

- **SSR**: server-side rendering
- **SSG**: static-site generation
- **ISR**: incremental static regeneration

### SSR/SSG/ISR

Next.jsは元々、SSRができるReactフレームワークとして2016/10に登場しました。以下は当時のVercel（旧Zeit）のアナウンス記事です。

https://vercel.com/blog/next

上記v1のアナウンスから長い間Next.jsはSSRをするためのフレームワークでしたが、約3年半後に登場する[v9.3](https://nextjs.org/blog/next-9-3)でSSG、[v9.5](https://nextjs.org/blog/next-9-5)でISRが導入されたことでNext.jsは複数のpre-rendering方式をサポートするフレームワークとなりました。

ここからは筆者の主観も含みます。当時は[Gatsby](https://www.gatsbyjs.com/)の台頭もありSSG人気が根強く、SSRしかできなかったNext.jsのユーザーはGatsbyに流れることも多かったように思います。実際、当時の筆者はNext.jsよりGatsbyを好んで使っていました。しかしNext.js v9系で需要の多かったdynamicルーティングやGatsbyが弱かったTypeScript対応などの実装、そして上記SSGやISRのサポートによりNext.jsは一気に注目を集めるようになりました。筆者にはこのv9系で一気に実装された機能群が、 今日のNext.jsの人気に繋がっているように思えるのです。

それだけpre-rendering方式は多くのユーザーの関心を集め、フレームワークの人気を支える重要な要素となる機能なのです。

### App Routerの登場以降

さて、上記のv9時点ではNext.jsはいわゆるPages Routerしか存在しませんでした。その後[v13](https://nextjs.org/blog/next-13)で発表されたApp Routerでは、RSCやServer Actions・多層のキャッシュなど多くのパラダイムシフトが必要となりました。App Routerでは、pre-rendering方式に関してはどのような変化があったのでしょうか？

結論から言うとApp Routerでも従来のSSR/SSG/ISR相当の機能をサポートしていますが、App Routerのドキュメントでは**SSR/SSG/ISRなどの用語は基本的に使用されていません**。

これらの単語が使われない理由について明確な説明を見つけることはできませんでしたが、筆者には大きく2つの理由が思い当たります。1つはRSCとSSRという概念が混在することで混乱を招く可能性があったこと、もう1つはISRが概念自体難しいと評されていたことです。ISRが難しいと評されていたのは筆者の観測範囲なので日本国内だけなのかもしれませんが、いづれにせよApp RouterはSSR/SSG/ISRという概念の理解を求めるのをやめたようです。

### static/dynamic rendering

現在App RouterはSSR/SSG/ISRではなく、[static rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default)と[dynamic rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)という2つの概念を使って多くの機能を説明しています。

static renderingは従来におけるSSGやISR相当で、build時やbackgroundでレンダリングされます。dynamic renderingはリクエストごとにレンダリングされる、従来のSSR相当のrenderingです。

ISRは後発だったのでマーケティング的にも機能名称が必要だったのかもしれませんが、整理するとこのstatic/dynamic renderingの方が概念としてはとても分かりやすいように筆者は感じています。

### 現状のNext.jsの問題点

さて、だいぶ整理されたpre-renderingですが、ユーザーからのフィードバックとして以下のようなものがあったそうです。

- ランタイム・設定・レンダリング方法など、考慮事項が多すぎる
- 静的化のメリットは享受したい
- 動的レンダリングもサポートしたい

これらを経て、よりよいパフォーマンスの達成とトレードオフに複雑さを強いないことを目標に開発されたのが本稿の主題であるPPRです。

## PPRとは

- SSR/SSG/ISRのいいとこどり

## PPRのモチベーション

## PPRの登場によって変わるpre-renderingの歴史

- 現状筆者には理想的なpre-rendering方式に思える
- Next.jsは今後この形になっていく
- Remixなどの他フレームワークはどうなんだろう
- Reactが最近大事にしてる並行性やStreamなどとの相性も良さそうだし、難しいかもしれないがPPRなフレームワークがもっと出てきてもいい気がする

## PPRの注意点

- 画面の静的化された部分については返してしまうため、ページは必ず200
- SEOに関係する部分はsuspendすると影響があるかもしれない

## 感想
