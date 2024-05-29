---
title: "PPR - pre-rendering新時代の到来とSSR/SSG論争の終焉"
emoji: "👑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

**Partial Pre-Rendering**(以降PPR)はNext.js v14.0で発表された、SSRやSSGにならぶ**新たなpre-rendering方式**です。

https://nextjs.org/blog/next-14#partial-prerendering-preview

PPRは本稿執筆時点の2024/05現在も開発中で、v15のRC版にてexperimentalフラグを有効にすることで利用することができます。`ppr: true`とすれば全部のページが対象となり、`ppr: "incremental"`とすれば`export const experimental_ppr = true`を設定したRoute SegmentのみがPPRの対象となります。

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

PPRはNext.jsコアチームにとっても重大な機能開発であり、個人的にはとても注目度の高いトピックなのですが、筆者の観測範囲では話題になってるのは一部でそこまで盛り上がってないように感じます。

筆者はPPRによってpre-renderingの時代がまた1つ新しいものになると考えています。本稿ではPPRとは何か、何を解決しようとしているのか、そしてPPR時代の到来によって何が変わるのか考察したいと思います。

## pre-renderingを振り返る

PPRの話をする前に、これまでのNext.jsのpre-renderingについて振り返ってみましょう。これまで、Next.jsがサポートしているpre-rendering方式は3つありました。

- **SSR**: server-side rendering
- **SSG**: static-site generation
- **ISR**: incremental static regeneration

これらがサポートされた歴史的経緯を筆者なりに振り返ってみます。

### Pages Router時代

Next.jsは元々、SSRができるReactフレームワークとして2016/10に登場しました。以下は当時のVercel（旧Zeit）のアナウンス記事です。

https://vercel.com/blog/next

上記v1のアナウンスから長い間Next.jsはSSRをするためのフレームワークでしたが、約3年半後に登場する[v9.3](https://nextjs.org/blog/next-9-3)でSSG、[v9.5](https://nextjs.org/blog/next-9-5)でISRが導入されたことでNext.jsは複数のpre-rendering方式をサポートするフレームワークとなりました。

当時は[Gatsby](https://www.gatsbyjs.com/)の台頭もありSSG人気が根強く、SSRしかできなかったNext.jsのユーザーはGatsbyに流れることも多かったように思います。実際、当時の筆者はNext.jsよりGatsbyを好んで使っていました。しかしNext.js v9系で需要の多かったdynamicルーティングやGatsbyが弱かったTypeScript対応などの実装、そして上記SSGやISRのサポートによりNext.jsは一気に注目を集めるようになりました。筆者にはこのv9系で実装された機能群が、 今日のNext.jsの人気に繋がったように思えます。

SSRかSSGかというpre-rendering方式の議論は多くのユーザーの関心を集め、これらをどちらもサポートしたNext.jsの選択が、今日の人気を支える重要な要素となっているのです。

### App Router登場以降

上記v9時点では、Next.jsはいわゆるPages Routerしか存在しませんでした。その後[v13](https://nextjs.org/blog/next-13)で発表されたApp Routerでは、RSC(React Server Components)やServer Actions・多層のキャッシュなど多くのパラダイムシフトが必要となりました。App Routerでは、pre-rendering方式に関してはどのような変化があったのでしょうか？

結論から言うとApp Routerは**従来同様SSR/SSG/ISR相当の機能をサポート**していますが、App Routerのドキュメントでは基本的にSSR/SSG/ISRなどの**用語は使用されていません**。

これらの用語が使われない理由について明確な説明を見つけることはできませんでしたが、筆者には大きく2つの理由が思い当たります。1つはRSCとSSRという概念がドキュメント上で同居すると混乱を招く可能性があったこと、もう1つはISRが概念自体難しいと評されていたことです(ISRが難しいと評されていたのは筆者の観測範囲なので日本国内だけなのかもしれませんが)。

### static renderingとdynamic rendering

現在App RouterはSSR/SSG/ISRではなく、[static rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default)と[dynamic rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)という2つの概念を使って多くの機能を説明しています。

**static rendering**は従来におけるSSGやISR相当で、build時やrevalidate実行後にレンダリングされます。**dynamic rendering**はリクエストごとにレンダリングされる、従来のSSR相当のrenderingです。

ISRは後発だったのでマーケティング的にも機能名称が必要だったのかもしれませんが、整理するとこのstatic/dynamic renderingの方が概念としてはとても分かりやすいように筆者は感じています。

### 現状のNext.jsの問題点

さて、だいぶ整理されたpre-rendering方式ですが、現状のNext.jsユーザーからのフィードバックとして以下のようなものがあったそうです。

- Next.jsはランタイム・設定・レンダリング方法など、考慮事項が多すぎる
- 静的化による速度・信頼性は重要
- 一方でリクエストごとの動的レンダリングのも重要

これらのフィードバックに同時に答えられるような、静的化のメリットを享受しつつ動的レンダリングにも対応し、シンプルな設計を目指すという意欲的な取り組みがなされたのが、本稿の主題であるPPRです。

## PPRとは

PPRは**static renderingしつつ、部分的にdynamic renderingにする**ことが可能なpre-renderingです。SSG・ISRのページの一部にSSRな部分を組み合わせたようなイメージが近いかもしれません。[公式の説明](https://rc.nextjs.org/learn/dashboard-app/partial-prerendering#what-is-partial-prerendering)よりECサイトの商品ページ例を拝借すると、以下のような構成が可能になります。

![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

商品ページ全体やナビゲーションはstatic renderingで静的化され、一方カートやレコメンド情報といったユーザーごとに異なるUI部分はdynamic renderingにすることができます。もちろん商品情報自体が更新されることもあるでしょうが、この例では必要に応じてrevalidateすることを想定しています。

dynamic renderingな部分は**dynamic hole**(もしくはasync holeやただのhole)などと呼ばれ、`Suspense`によって境界を定義することができます。現状experimentalなので冒頭述べた設定は必要ですが、他に**新たなAPIを学習する必要はありません**。

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

上記の実装例では`<ShoppingCart>`と`<Recommendations>`はdynamic renderingである必要があり、`<Suspense>`の`fallback`にこれらのレンダリング中に表示するスケルトンUIを指定しています。このコンポーネントはPPRされると、以下のようなhtmlを生成します。

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

初期表示には上記のDOMが利用され、`<ShoppingCart>`や`<Recommendations>`のレンダリングが終わり次第Streamを介してクライアントサイドに送信されdynamic holeのスケルトンUIを置き換えます。これらが**1つのhttpリクエスト内で完結する**のも大きな特徴です。

### PPRの観察

実際にPPRによってdynamic holeが置き換わる様子を観察してみましょう。

Next.jsのレスポンスはStreamになっており、PPRにおいてはstatic renderingな部分をまず返却します。その後dynamic renderingが完了したらStreamを介してクライアントに送信され、dynamic holeの部分を置き換えます。

以下のサンプルコードを元に、挙動を観察してみます。

```tsx
// app/ppr/page.tsx
import { Suspense } from "react";
import { setTimeout } from "node:timers/promises";

export default function Home() {
  return (
    <main>
      <h1>PPR Page</h1>
      <Suspense fallback={<>loading...</>}>
        <RandomTodo />
      </Suspense>
    </main>
  );
}

async function RandomTodo() {
  // use dynamic function
  const todo = await fetch("https://dummyjson.com/todos/random", {
    // v15.0.0-rc.0時点ではデフォルトでno-storeだが、明示的に指定しないとdynamic renderingにならない
    cache: "no-store",
  }).then((res) => res.json());
  await setTimeout(3000);

  console.log("todo on ppr", todo);

  return (
    <>
      <h2>Random Todo</h2>
      <code>
        <pre>{JSON.stringify(todo, null, 2)}</pre>
      </code>
    </>
  );
}

export const experimental_ppr = true;
```

`RandomTodo`はリクエストの度にランダムなTODO情報を取得するコンポーネントです。今回はStreamの様子を観察したいので、あえて3秒遅延させています。

:::message
今回の主題ではないのですが、v15.0.0-rc.0時点ではデフォルトでfetchは`no-store`ですが、**明示的に指定しないとdynamic renderingにならない**という仕様になっているので明示的に指定しています。同様の対策として[`unstable_noStore`](https://nextjs.org/docs/app/api-reference/functions/unstable_noStore)を使ってもいいかもしれません。
:::

このページの表示とレスポンスの様子は以下のようになります。

_初期描画_
![ppr stream start](/images/nextjs-partial-pre-rendering/ppr-stream-start.png)

_dynamic rendering完了後_
![ppr stream end](/images/nextjs-partial-pre-rendering/ppr-stream-end.png)

初期描画時はhtmlのレスポンスが途中までしか帰ってきておらず、`loading...`が表示されています。3秒後にdynamic renderingが完了するとStreamを介してクライアントに送信され、`loading...`が`Random Todo`に置き換わります。実際のレスポンスのhtmlに含まれる`<body>`を見てみます。`<div hidden id="S:0">`以降が遅れて送信されてくるレスポンスです。

```html
<body>
<main>
    <h1>PPR Page</h1>
    <!--$?-->
    <template id="B:0"></template>
    loading...
    <!--/$-->
</main>
<script src="/_next/static/chunks/webpack-b5d81ab04c5b38dd.js" async=""></script>
<div hidden id="S:0">
    <h2>Random Todo</h2>
    <code>
                <pre>{
  &quot;id &quot;: 138,
  &quot;todo &quot;: &quot;Compliment someone &quot;,
  &quot;completed &quot;: false,
  &quot;userId &quot;: 76
}</pre>
    </code>
</div>
<script>
    $RC = function (b, c, e) {
        c = document.getElementById(c);
        c.parentNode.removeChild(c);
        var a = document.getElementById(b);
        if (a) {
            b = a.previousSibling;
            if (e)
                b.data = "$!",
                        a.setAttribute("data-dgst", e);
            else {
                e = b.parentNode;
                a = b.nextSibling;
                var f = 0;
                do {
                    if (a && 8 === a.nodeType) {
                        var d = a.data;
                        if ("/$" === d)
                            if (0 === f)
                                break;
                            else
                                f--;
                        else
                            "$" !== d && "$?" !== d && "$!" !== d || f++
                    }
                    d = a.nextSibling;
                    e.removeChild(a);
                    a = d
                } while (a);
                for (; c.firstChild;)
                    e.insertBefore(c.firstChild, a);
                b.data = "$"
            }
            b._reactRetry && b._reactRetry()
        }
    }
    ;
    $RC("B:0", "S:0")
</script>
<script>
    (self.__next_f = self.__next_f || []).push([0]);
    self.__next_f.push([2, null])
</script>
<script>
    self.__next_f.push([1, "1:I[4129,[],\"\"]\n3:\"$Sreact.suspense\"\n5:I[8330,[],\"\"]\n6:I[3533,[],\"\"]\n8:I[6344,[],\"\"]\n9:[]\n"])
</script>
<script>
    self.__next_f.push([1, "0:[null,[\"$\",\"$L1\",null,{\"buildId\":\"nEXD0Hu3p8AtFQuHj1ZPn\",\"assetPrefix\":\"\",\"initialCanonicalUrl\":\"/ppr\",\"initialTree\":[\"\",{\"children\":[\"ppr\",{\"children\":[\"__PAGE__\",{}]}]},\"$undefined\",\"$undefined\",true],\"initialSeedData\":[\"\",{\"children\":[\"ppr\",{\"children\":[\"__PAGE__\",{},[[\"$L2\",[\"$\",\"main\",null,{\"children\":[[\"$\",\"h1\",null,{\"children\":\"PPR Page\"}],[\"$\",\"$3\",null,{\"fallback\":\"loading...\",\"children\":\"$L4\"}]]}]],null],null]},[\"$\",\"$L5\",null,{\"parallelRouterKey\":\"children\",\"segmentPath\":[\"children\",\"ppr\",\"children\"],\"error\":\"$undefined\",\"errorStyles\":\"$undefined\",\"errorScripts\":\"$undefined\",\"template\":[\"$\",\"$L6\",null,{}],\"templateStyles\":\"$undefined\",\"templateScripts\":\"$undefined\",\"notFound\":\"$undefined\",\"notFoundStyles\":\"$undefined\",\"styles\":null}],null]},[[\"$\",\"html\",null,{\"lang\":\"en\",\"children\":[\"$\",\"body\",null,{\"children\":[\"$\",\"$L5\",null,{\"parallelRouterKey\":\"children\",\"segmentPath\":[\"children\"],\"error\":\"$undefined\",\"errorStyles\":\"$undefined\",\"errorScripts\":\"$undefined\",\"template\":[\"$\",\"$L6\",null,{}],\"templateStyles\":\"$undefined\",\"templateScripts\":\"$undefined\",\"notFound\":[[\"$\",\"title\",null,{\"children\":\"404: This page could not be found.\"}],[\"$\",\"div\",null,{\"style\":{\"fontFamily\":\"system-ui,\\\"Segoe UI\\\",Roboto,Helvetica,Arial,sans-serif,\\\"Apple Color Emoji\\\",\\\"Segoe UI Emoji\\\"\",\"height\":\"100vh\",\"textAlign\":\"center\",\"display\":\"flex\",\"flexDirection\":\"column\",\"alignItems\":\"center\",\"justifyContent\":\"center\"},\"children\":[\"$\",\"div\",null,{\"children\":[[\"$\",\"style\",null,{\"dangerouslySetInnerHTML\":{\"__html\":\"body{color:#000;background:#fff;margin:0}.next-error-h1{border-right:1px solid rgba(0,0,0,.3)}@media (prefers-color-scheme:dark){body{color:#fff;background:#000}.next-error-h1{border-right:1px solid rgba(255,255,255,.3)}}\"}}],[\"$\",\"h1\",null,{\"className\":\"next-error-h1\",\"style\":{\"display\":\"inline-block\",\"margin\":\"0 20px 0 0\",\"padding\":\"0 23px 0 0\",\"fontSize\":24,\"fontWeight\":500,\"verticalAlign\":\"top\",\"lineHeight\":\"49px\"},\"children\":\"404\"}],[\"$\",\"div\",null,{\"style\":{\"display\":\"inline-block\"},\"children\":[\"$\",\"h2\",null,{\"style\":{\"fontSize\":14,\"fontWeight\":400,\"lineHeight\":\"49px\",\"margin\":0},\"children\":\"This page could not be found.\"}]}]]}]}]],\"notFoundStyles\":[],\"styles\":null}]}]}],null],null],\"couldBeIntercepted\":false,\"initialHead\":[false,\"$L7\"],\"globalErrorComponent\":\"$8\",\"missingSlots\":\"$W9\"}]]\n"])
</script>
<script>
    self.__next_f.push([1, "a:\"$Sreact.fragment\"\n7:[\"$\",\"$a\",\"eZy4y-XpmIcc-gbnXZore\",{\"children\":[[\"$\",\"meta\",\"0\",{\"name\":\"viewport\",\"content\":\"width=device-width, initial-scale=1\"}],[\"$\",\"meta\",\"1\",{\"charSet\":\"utf-8\"}],[\"$\",\"title\",\"2\",{\"children\":\"Create Next App\"}],[\"$\",\"meta\",\"3\",{\"name\":\"description\",\"content\":\"Generated by create next app\"}],[\"$\",\"link\",\"4\",{\"rel\":\"icon\",\"href\":\"/favicon.ico\",\"type\":\"image/x-icon\",\"sizes\":\"16x16\"}]]}]\n2:null\n"])
</script>
<script>
    self.__next_f.push([1, "4:[[\"$\",\"h2\",null,{\"children\":\"Random Todo\"}],[\"$\",\"code\",null,{\"children\":[\"$\",\"pre\",null,{\"children\":\"{\\n  \\\"id\\\": 138,\\n  \\\"todo\\\": \\\"Compliment someone\\\",\\n  \\\"completed\\\": false,\\n  \\\"userId\\\": 76\\n}\"}]}]]\n"])
</script>
</body>
```

注目すべきはscriptの`$RC`らへんです。dynamic holeの部分にあるtemplateのidが`B:0`、Streamを介して後半送られてきたdynamic renderingのDOMが`S:0`、これら`$RC("B:0", "S:0")`で交換しているのがわかります。

また、scriptが直接記述されてることからも前述の通りこれらが**1つのhttpリクエスト内で完結**していることもわかります。最初筆者はPPRの仕組みについて、Suspenseを利用してるしRender-as-you-fetchしてるのかと思ってたのですが、それすらなく1つのレスポンス内でこれらが完結するようになっているのは驚きました。比較実験してないので筆者の理解範囲ですが、遅延表示するケースにおいてこの実装は理論上非常に高速なのではないかと推測できます。

## SSR/SSG論争の終焉とPPR後の議論

PPR前後で我々Next.jsユーザーに必要になるパラダイムシフトについても考察したいと思います。前述の例のように、ページの一部をAPIを介して動的にしたいケースについて考えてみます。PPR以前なら、このような画面を実装するのに2つの選択肢がありました。

- SSG+CSR fetchで一部をクライアントサイドで動的にする
- SSRでページ全体をサーバーサイドで動的にする

これらの選択肢にはそれぞれ懸念が存在します。

SSG+CSR fetch(Render-as-you-fetchやFetch-on-render)の場合、レンダリング後にfetchが発生しラウンドトリップが1回増えることでパフォーマンス的に不利になりえます。また、レンダリング後にfetchする実装や、fetch先のエンドポイントの実装などが必要になるため実装コストも増えます。

一方ページ全体をSSRするようした場合は実装はシンプルになりますが、サーバー側でhttpリクエストのウォーターフォールが発生してしまい、ページ全体のレスポンスが遅くなってしまいます。場合によってはページのごく一部（前述の例におけるカートなど）の表示のために、ページ全体が遅くなってしまうことも往々にしてあるでしょう。

これら2つの選択肢にはトレードオフが付き纏っていたため、SSRにすべきかSSG(+CSR fetch)にすべきか、ケースバイケースなため論争になりがちでした。PPRはシンプルな実装、非ウォーターフォールなレスポンス、1つのhttpリクエスト内で完結するという特徴を兼ね備えています。PPR以降、トレードオフを意識して選択する必要のあったこのSSR/SSG論争は不要となり、代わりに「**どこまでをstaticに、どこまでをdynamicにするか**」という議論が重要になってきます。

## PPRの注意点

ここまでPPRを銀の弾丸のように述べてきましたが、PPRにも当然注意点があります。筆者が思う注意点をいくつか紹介したいと思います。

### SEO的に重要な要素は遅延させるべきではない

1つは`<Suspense>`で遅延レンダリングさせた部分は初期のDOMには存在しないため、**SEO影響がある可能性がある**ということです。

前述のような商品ページの例においてカートやレコメンド情報を遅延レンダリングさせてもSEO影響はほぼないでしょう(レコメンドから内部リンクが伸びてるであろうことはこの際無視できるものとします)。しかし商品情報を遅延レンダリングさせてしまうと、JSが実行されないとDOM上に商品情報がないままになってしまいます。

GoogleのクローラーにはJSを実行せずクローリングするケースとJSを実行しつつクローリングするケースがあります。筆者にはこれらが実態として別物なクローラーなのかどうかはわかりませんが、JSを実行するクローラーの巡回はJSを実行しないクローラーに比べて訪問が遅くなる傾向にあるという話です。

:::message
クローラーについては情報がブラックボックスになりがちなこともあり、軽く調べた程度ではエビデンスらしくものは見つけられませんでした。筆者が過去に聞いた話なので情報が古い・誤ってるなどあればご指摘下さい。
:::

### ページは必ず200になる

もう1つはPPRは画面の静的化された部分については返してしまうため、**ページのhttp statusは必ずOK(200)になってしまう**と言う点です。

上記の懸念を考慮してSEOに影響のある部分についてはstatic renderingしてるものとした場合、これが実害になってくるのは監視周りでしょう。そもそもApp Routerを利用してる場合にはStreamによるレスポンスがベースとなるため、http statusで監視するだけでは不完全です。なので改めてになりますが、App Routerで実装したアプリケーションの監視はhttp statusではなく個別のエラー発生率などを元に行う必要があるでしょう。

### static renderingで完結できるならその方が良い

最後に、PPRはあくまでdynamic renderingを必要とするケースにおける最適化である、ということを念頭におく必要があります。Xでやりとりしてて以下のようなコメントをいただきました。

https://twitter.com/sumiren_t/status/1793620259586666643

> 一部を静的化できる＝SSRでもSGと同じだけ速い、みたいな勘違いがされてないといいなという話でした（自分も前はそういう勘違いをしていたので）

dynamic renderingを含むページでも、PPRならTTFB(Time to First Bytes)をSSG相当にすることが可能です。しかし、パフォーマンスにおける「速い」というのは計測するものによって異なるので、PPRなら一律速くなるというわけではありません。例えばTTI(Time to Interactive)においては、遅延レンダリングの部分がinteractiveになるまでの時間はSSRとそう変わらないでしょう。

この点は確かに混乱しやすいところかもしれません。PPRにおけるパフォーマンスの話をする際には、どの速度指標における話なのか注意して話す必要があるでしょう。

### RC版のexperimental段階である

PPRは本稿執筆現在、v15のRC版でexperimentalで提供されています。筆者が試した限りでは壊滅的に動かないといったことはなさそうですが、まだまだ使用してるユーザーが少なく、潜在的なバグもまだ予想されます。

現段階では実験的プロダクトやlocal環境でのみ利用するのが良いでしょう。

## 感想

PPRは現存するpre-rendering方式において最も理想的でシンプルな設計ではないかと筆者は感じています。もちろん裏側では非常に複雑なことをやっているわけですが、我々Next.jsユーザーからすれば「基本はstatic rendering、一部をdynamic rendering」というシンプルなルールに従うのみです。

RSC以降のReactでは、データフェッチをはじめとしたサーバー側処理もコンポーネント責務にするなど、「必要なことは全てコンポーネントにカプセル化する」という方向性が強まっているように感じます。そしてレンダリングに境界を設け並行性を高めるのが`<Suspense>`です。これらを鑑みても、PPRは非常に昨今のReactらしい設計と言えるのではないでしょうか。
