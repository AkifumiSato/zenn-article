---
title: "PPRに見る新時代pre-rendering - SSR/SSG論争の終焉"
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

// page.tsx(layout.tsxでも可)
export default function Page() {
  // ...
}

export const experimental_ppr = true
```

PPRはNext.js開発チームにとっても重大な機能開発であり、個人的にはとても注目度の高いトピックなのですが、筆者の観測範囲では話題になってるのは一部でそこまで盛り上がってないように感じます。

本稿を通じてPPRとは何か、何を解決しようとしているのか、そしてPPRが重大な機能開発たる所以をお伝えできればと思います。

## pre-renderingを振り返る

PPRの話をする前に、これまでのNext.jsのpre-renderingについて振り返ってみましょう。これまで、Next.jsがサポートしているpre-rendering方式は3つありました。

- **SSR**: server-side rendering
- **SSG**: static-site generation
- **ISR**: incremental static regeneration

### Pages Router時代

Next.jsは元々、SSRができるReactフレームワークとして2016/10に登場しました。以下は当時のVercel（旧Zeit）のアナウンス記事です。

https://vercel.com/blog/next

上記v1のアナウンスから長い間Next.jsはSSRをするためのフレームワークでしたが、約3年半後に登場する[v9.3](https://nextjs.org/blog/next-9-3)でSSG、[v9.5](https://nextjs.org/blog/next-9-5)でISRが導入されたことでNext.jsは複数のpre-rendering方式をサポートするフレームワークとなりました。

ここからは筆者の主観も含みます。当時は[Gatsby](https://www.gatsbyjs.com/)の台頭もありSSG人気が根強く、SSRしかできなかったNext.jsのユーザーはGatsbyに流れることも多かったように思います。実際、当時の筆者はNext.jsよりGatsbyを好んで使っていました。しかしNext.js v9系で需要の多かったdynamicルーティングやGatsbyが弱かったTypeScript対応などの実装、そして上記SSGやISRのサポートによりNext.jsは一気に注目を集めるようになりました。筆者にはこのv9系で一気に実装された機能群が、 今日のNext.jsの人気に繋がっているようにさえ思えます。

SSRかSSGかというpre-rendering方式の議論は多くのユーザーの関心を集め、今日のNext.jsの人気を支える重要な要素となっている機能なのです。

### App Router登場以降

さて、上記のv9時点ではNext.jsはいわゆるPages Routerしか存在しませんでした。その後[v13](https://nextjs.org/blog/next-13)で発表されたApp Routerでは、RSC(React Server Components)やServer Actions・多層のキャッシュなど多くのパラダイムシフトが必要となりました。App Routerでは、pre-rendering方式に関してはどのような変化があったのでしょうか？

結論から言うとApp Routerは従来同様SSR/SSG/ISR相当の機能をサポートしていますが、App Routerのドキュメントでは**SSR/SSG/ISRなどの用語は基本的に使用されていません**。

これらの単語が使われない理由について明確な説明を見つけることはできませんでしたが、筆者には大きく2つの理由が思い当たります。1つはRSCとSSRという概念が混在することで混乱を招く可能性があったこと、もう1つはISRが概念自体難しいと評されていたことです(ISRが難しいと評されていたのは筆者の観測範囲なので日本国内だけなのかもしれませんが)。

現在App RouterはSSR/SSG/ISRではなく、[static rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default)と[dynamic rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)という2つの概念を使って多くの機能を説明しています。

**static rendering**は従来におけるSSGやISR相当で、build時やrevalidate実行後にレンダリングされます。**dynamic rendering**はリクエストごとにレンダリングされる、従来のSSR相当のrenderingです。

ISRは後発だったのでマーケティング的にも機能名称が必要だったのかもしれませんが、整理するとこのstatic/dynamic renderingの方が概念としてはとても分かりやすいように筆者は感じています。

### 現状のNext.jsの問題点

さて、だいぶ整理されたpre-rendering方式ですが、現状のNext.jsユーザーからのフィードバックとして以下のようなものがあったそうです。

- Next.jsはランタイム・設定・レンダリング方法など、考慮事項が多すぎる
- 静的化による速度・信頼性は重要
- 一方でリクエストごとの動的レンダリングのも重要

これらのフィードバックを得て、よりよいパフォーマンスの達成のトレードオフで複雑さを強いることのないよう設計されたのが、本稿の主題であるPPRです。

## PPRとは

PPRはstatic renderingしつつ、部分的にdynamic renderingにすることが可能にする技術です。SSG・ISRのページの一部にSSRな部分を組み合わせたようなイメージが近いかもしれません。[公式の説明](https://rc.nextjs.org/learn/dashboard-app/partial-prerendering#what-is-partial-prerendering)よりECサイトの商品ページを例にすると、以下のような構成が可能になります。

![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

商品ページ全体やナビゲーションはstatic renderingで静的化され、一方カートやレコメンド情報といったユーザーごとに異なるUI部分はdynamic renderingにすることができます。このdynamic renderingな部分は**dynamic hole**(もしくはasync hole)などと呼ばれ、`Suspense`によって境界を定義することができます。

Next.jsのPPRではまずレスポンスのStreamにstatic renderingされたhtml部分から即座にクライアントに返し始めます。App Routerでは初期ロードのhtml要素はstatic renderingの結果が反映されているおで、初期表示においてはholeの部分には`Suspense`に指定したloaderなどの`fallback`が表示されることになります。レスポンスの最後の方(htmlの最後の方)でRSC Payloadを含む部分が返され、ここにdynamic renderingのRSC Payloadも含まれます。そのため1つのリクエストの前半パートでstaticな部分を返してユーザーに表示、後半パートでdynamicな部分を徐々に返すという手法をとることで1つのhttpリクエストで静的・動的なレンダリングを混在させつつ高速なレスポンスを実現しています。

Todo: ↑の説明書き直した方がいいかも

実際にPPRで境界を定義づけるのは`Suspense`です。現状experimentalなので前述の設定などは必要ですが、他に**新たなAPIを学ぶ必要はありません**。

```tsx
export default function Page() {
  return (
    <main>
      <header>
        <h1>My Store</h1>
        <Suspense fallback={<CartSkeleton />}>
          <ShoppingCart />
        </Suspense>
      </header>
      <Banner />
      <Suspense fallback={<ProductListSkeleton />}>
        <Recommendations />
      </Suspense>
      <NewProducts />
    </main>
  );
}
```

上記の実装例では`<ShoppingCart>`と`<Recommendations>`はdynamic renderingである必要があり、`Suspense`の`fallback`にこれらのレンダリング中に表示するスケルトンUIを指定しています。このコンポーネントはPPRされると、以下のようなhtmlを生成します。

```html
<main>
  <header>
    <h1>My Store</h1>
    <div class="cart-skeleton">
      <!-- Hole -->
    </div>
  </header>
  <div class="banner" />
  <div class="product-list-skeleton">
    <!-- Hole -->
  </div>
  <section class="new-products" />
</main>
```

初期表示には上記のhtmlが返されて`<ShoppingCart>`や`<Recommendations>`のレンダリングが終わり次第、Streamを介してクライアントサイドに送信されdynamic holeのスケルトンUIを置き換えます。

## SSR/SSG論争

また筆者の主観を含みますが、PPR以前は「SSRとSSGどちらにすべきか」というような対立的選論争が起きがちでした。

TBW: SSR/SSGのユースケースについて

ISRはどうなのか気になるかもしれませんが、ISRは概念自体が難しいのかキャッシュのハンドリング観点からVercel以外だと使いづらいからなのか、筆者の周りだと利用してないというチームの方が多かったです。この辺は昨今Cache Handlerが設定できるようになったことにより、セルフホスティング環境でも使いやすくなったかもしれません。

https://zenn.dev/akfm/articles/nextjs-cache-handler-redis

- 現状筆者には理想的なpre-rendering方式に思える
- Next.jsは今後この形になっていく
- Remixなどの他フレームワークはどうなんだろう
- Reactが最近大事にしてる並行性やStreamなどとの相性も良さそうだし、難しいかもしれないがPPRなフレームワークがもっと出てきてもいい気がする

## PPRの注意点

- 画面の静的化された部分については返してしまうため、ページは必ず200
- SEOに関係する部分はsuspendすると影響があるかもしれない

## 感想
