---
title: "PPR - pre-rendering新時代の到来とSSR/SSG論争の終焉"
emoji: "👑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: true
---

:::message alert
本稿は Next.js v15.0.0-rc.0 時点の情報を元に執筆しており、PPR はさらに experimental な機能です。v15.0.0 のリリース時や、PPR が stable な機能として提供される際には機能の一部が変更されてる可能性がありますので、ご注意下さい。
:::

**Partial Pre-Rendering**(以降 PPR)は Next.js v14.0 で発表された、SSR や SSG にならぶ**新たな pre-rendering 方式**です。

https://nextjs.org/blog/next-14#partial-prerendering-preview

PPR は前述の通り開発中の機能で、v15 の RC 版にて experimental フラグを有効にすることで利用することができます。
`ppr: true`とすれば全部のページが対象となり、`ppr: "incremental"`とすれば`export const experimental_ppr = true`を設定した Route Segment のみが PPR の対象となります。

https://rc.nextjs.org/docs/app/api-reference/next-config-js/ppr

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    ppr: "incremental", // ppr: boolean | "incremental"
  },
};

module.exports = nextConfig;

// page.tsx(layout.tsxでも可)
export const experimental_ppr = true;

export default function Page() {
  // ...
}
```

PPR は Next.js コアチームにとっても重大な機能開発であり、個人的にはとても注目度の高いトピックなのですが、筆者の観測範囲では話題になってるのは一部でそこまで盛り上がってないように感じます。

筆者は PPR によって pre-rendering の時代がまた 1 つ新しいものになると考えています。本稿では PPR とは何か、何を解決しようとしているのか、そして PPR 時代の到来によって何が変わるのか考察したいと思います。

## pre-rendering を振り返る

PPR の話をする前に、これまでの Next.js の pre-rendering について振り返ってみましょう。これまで、Next.js がサポートしている pre-rendering 方式は 3 つありました。

- **SSR**: server-side rendering
- **SSG**: static-site generation
- **ISR**: incremental static regeneration

これらがサポートされた歴史的経緯を筆者なりに振り返ってみます。

### Pages Router 時代

Next.js は元々、SSR ができる React フレームワークとして 2016/10 に登場しました。以下は当時の Vercel（旧 Zeit）のアナウンス記事です。

https://vercel.com/blog/next

上記 v1 のアナウンスから長い間 Next.js は SSR をするためのフレームワークでしたが、約 3 年半後の2019年に登場する[v9.3](https://nextjs.org/blog/next-9-3)で SSG、[v9.5](https://nextjs.org/blog/next-9-5)で ISR が導入されたことで Next.js は複数の pre-rendering 方式をサポートするフレームワークとなりました。

当時は[Gatsby](https://www.gatsbyjs.com/)の台頭もあり SSG 人気が根強く、SSR しかできなかった Next.js のユーザーは Gatsby に流れることも多かったように思います。

_2019 年ごろは Gatsby の方が上回っている(見づらくてすいません)_
![npm trends](/images/nextjs-partial-pre-rendering/npm-trends.png)

実際、当時の筆者は好んで Gatsby を使っていました。しかし Next.js v9 系で需要の多かった dynamic ルーティングや Gatsby が弱かった TypeScript 対応などの実装、そして上記 SSG や ISR のサポートにより Next.js は一気に注目を集めるようになりました。筆者にはこの v9 系で実装された機能群が、 今日の Next.js の人気に繋がったように思えます。

SSR か SSG かという pre-rendering 方式の議論は多くのユーザーの関心を集め、これらをどちらもサポートした Next.js の選択が、今日の人気を支える重要な要素となっているのです。

### App Router 登場以降

上記 v9 時点では、Next.js はいわゆる Pages Router しか存在しませんでした。その後[v13](https://nextjs.org/blog/next-13)で発表された App Router では、RSC(React Server Components)や Server Actions・多層のキャッシュなど多くのパラダイムシフトが必要となりました。App Router では、pre-rendering 方式に関してはどのような変化があったのでしょうか？

結論から言うと App Router は従来同様**SSR/SSG/ISR 相当の機能をサポート**していますが、App Router のドキュメントでは基本的に SSR/SSG/ISR などの**用語は使用されていません**。

現在 App Router は SSR/SSG/ISR ではなく、[static rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default)と[dynamic rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)という 2 つの概念を使って多くの機能を説明しています。

- **static rendering**: 従来の SSG や ISR 相当で、build 時や revalidate 実行後にレンダリング
  - revalidate なし: SSG 相当
  - revalidate あり: ISR 相当
- **dynamic rendering**: 従来の SSR 相当で、リクエストごとにレンダリング

Pages Router では SSG か ISR かは build 時に実行する関数で設定する必要があったため静的に決定されていましたが、App Router における revalidate は`revalidatePath`や`revalidateTag`で動的に行うことができるので、SSG か ISR かは静的に決定されません。そのため Next.js からすると SSG と ISR を区別することに意味がなくなってしまったことが、これらの用語を使わなくなった理由かもしれません。

また、もう1つ理由として考えられるのは、ISRはVercel以外での運用が難しいと評される機能でネガティブなイメージがついてしまっていたことです。今日では**Cache Handler**によってキャッシュの永続化先を選択できるようになったため、ISR登場時よりはセルフホスティング環境などでも運用しやすくなったはずです。詳しくは筆者の過去記事をご参照ください。

https://zenn.dev/akfm/articles/nextjs-cache-handler-redis

### 現状の Next.js の問題点

さて、だいぶ整理された pre-rendering 方式ですが、現状の Next.js ユーザーからのフィードバックとして以下のようなものがあったそうです。

- Next.js はランタイム・設定・レンダリング方法など、考慮事項が多すぎる
- 静的化による速度・信頼性は重要
- 一方でリクエストごとの動的レンダリングのも重要

出典: https://nextjs.org/blog/next-14#motivation

これらのフィードバックに同時に答えられるような、静的化のメリットを享受しつつ動的レンダリングにも対応し、シンプルな設計を目指すという意欲的な取り組みがなされたのが、本稿の主題である PPR です。

## PPR とは

PPR は**static rendering しつつ、部分的に dynamic rendering にする**ことが可能な pre-rendering です。SSG・ISR のページの一部に SSR な部分を組み合わせられるようなイメージです。[公式の説明](https://rc.nextjs.org/learn/dashboard-app/partial-prerendering#what-is-partial-prerendering)より EC サイトの商品ページの例を拝借すると、以下のような構成が可能になります。

![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

商品ページ全体やナビゲーションは static rendering で静的化され、一方カートやレコメンド情報といったユーザーごとに異なる UI 部分は dynamic rendering にすることができます。もちろん商品情報自体が更新されることもあるでしょうが、この例では必要に応じて revalidate することを想定しています。

dynamic rendering な部分は**dynamic hole**(もしくは async hole やただの hole)などと呼ばれ、`<Suspense>`によって境界を定義することができます。現状 experimental なので冒頭述べた設定は必要ですが、他に**新たな API を学習する必要はありません**。

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

上記の実装例では`<ShoppingCart>`と`<Recommendations>`は dynamic rendering である必要があり、`<Suspense>`の`fallback`にこれらのレンダリング中に表示するスケルトン UI を指定しています。このコンポーネントは PPR されると、以下のような html を生成します。

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

初期表示には上記の DOM が利用され、これらは静的化されているので即座にリクエストもとに送信されます。その後、`<ShoppingCart>`や`<Recommendations>`のレンダリングが終わり次第 dynamic hole のスケルトン UI を置き換えます。これらが**1 つの http リクエスト内で完結する**のも大きな特徴です。

### PPR の観察

実際に PPR によって dynamic hole が置き換わる様子を観察してみましょう。

前述の通り Next.js のレスポンスは Stream になっており、PPR の場合はまず static rendering な部分を送信します。その後 dynamic rendering な部分が送信され、クライアントサイドの処理で dynamic hole を置き換えます。

以下のサンプルコードを元に、挙動を観察してみます。

```tsx
// app/ppr/page.tsx
import { Suspense } from "react";
import { setTimeout } from "node:timers/promises";

export const experimental_ppr = true;

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
  const todoDto: TodoDto = await fetch("https://dummyjson.com/todos/random", {
    // v15.0.0-rc.0時点ではデフォルトでno-storeだが、明示的に指定しないとdynamic renderingにならない
    cache: "no-store",
  }).then((res) => res.json());
  await setTimeout(3000);

  return (
    <>
      <h2>Random Todo</h2>
      <ul>
        <li>id: {todoDto.id}</li>
        <li>todo: {todoDto.todo}</li>
        <li>completed: {todoDto.completed ? "true" : "false"}</li>
        <li>userId: {todoDto.userId}</li>
      </ul>
    </>
  );
}

type TodoDto = {
  id: number;
  todo: string;
  completed: boolean;
  userId: number;
};
```

`RandomTodo`はリクエストの度にランダムな TODO 情報を取得するコンポーネントです。今回は Stream の様子を観察したいので、あえて 3 秒遅延させています。

:::message
今回の主題ではないのですが、v15.0.0-rc.0 時点ではデフォルトで fetch は`no-store`ですが、**明示的に指定しないと dynamic rendering にならない**という仕様になっているのでサンプルコードでは明示的に指定しています。同様の対策として[`unstable_noStore`](https://nextjs.org/docs/app/api-reference/functions/unstable_noStore)などの[dynamic functions](https://nextjs.org/docs/app/building-your-application/routing/route-handlers#dynamic-functions)を使ってもいいかもしれません。

この仕様については[RC 中に変更される可能性](https://x.com/feedthejim/status/1794778189354705190)が示唆されています。
:::

このページの表示とレスポンスの様子は以下のようになります。

_初期描画_
![ppr stream start](/images/nextjs-partial-pre-rendering/ppr-stream-start.png)

_dynamic rendering 完了後_
![ppr stream end](/images/nextjs-partial-pre-rendering/ppr-stream-end.png)

初期描画時は html のレスポンスが`<main>`の直後の`<script>`タグまでしか帰ってきておらず、画面上には`loading...`が表示されています。dynamic rendering が完了すると同じレスポンスでクライアントに残りの HTML が送信され、、それを受け取った Next.js が`loading...`を`Random Todo`に置き換えます。

実際に上記のサンプルコードを実行した時の html に含まれる`<body>`を見てみましょう。`<div hidden id="S:0">`以降が遅れて送信されてくる dynamic rendering の部分です。

```html
<body>
  <main>
    <h1>PPR Page</h1>
    <!--$?-->
    <template id="B:0"></template>
    loading...
    <!--/$-->
  </main>
  <script
    src="/_next/static/chunks/webpack-b5d81ab04c5b38dd.js"
    async=""
  ></script>
  <!-- 📍ここまでがstatic rendering、以降dynamic rendering -->
  <div hidden id="S:0">
    <h2>Random Todo</h2>
    <ul>
      <li>
        id:
        <!-- -->
        182
      </li>
      <li>
        todo:
        <!-- -->
        Explore a nearby trail
      </li>
      <li>
        completed:
        <!-- -->
        true
      </li>
      <li>
        userId:
        <!-- -->
        188
      </li>
    </ul>
  </div>
  <script>
    $RC = function (b, c, e) {
      c = document.getElementById(c);
      c.parentNode.removeChild(c);
      var a = document.getElementById(b);
      if (a) {
        b = a.previousSibling;
        if (e) (b.data = "$!"), a.setAttribute("data-dgst", e);
        else {
          e = b.parentNode;
          a = b.nextSibling;
          var f = 0;
          do {
            if (a && 8 === a.nodeType) {
              var d = a.data;
              if ("/$" === d)
                if (0 === f) break;
                else f--;
              else ("$" !== d && "$?" !== d && "$!" !== d) || f++;
            }
            d = a.nextSibling;
            e.removeChild(a);
            a = d;
          } while (a);
          for (; c.firstChild; ) e.insertBefore(c.firstChild, a);
          b.data = "$";
        }
        b._reactRetry && b._reactRetry();
      }
    };
    $RC("B:0", "S:0");
  </script>
  <script>
    (self.__next_f = self.__next_f || []).push([0]);
    self.__next_f.push([2, null]);
  </script>
  <script>
    self.__next_f.push([
      1,
      '1:I[4293,[],""]\n3:"$Sreact.suspense"\n5:I[1041,[],""]\n6:I[4762,[],""]\n8:I[2246,[],""]\n9:[]\n',
    ]);
  </script>
  <script>
    self.__next_f.push([
      1,
      '0:[null,["$","$L1",null,{"buildId":"6K-rjHj3iXikXC7ozNr2u","assetPrefix":"","initialCanonicalUrl":"/ppr","initialTree":["",{"children":["ppr",{"children":["__PAGE__",{}]}]},"$undefined","$undefined",true],"initialSeedData":["",{"children":["ppr",{"children":["__PAGE__",{},[["$L2",["$","main",null,{"children":[["$","h1",null,{"children":"PPR Page"}],["$","$3",null,{"fallback":"loading...","children":"$L4"}]]}]],null],null]},["$","$L5",null,{"parallelRouterKey":"children","segmentPath":["children","ppr","children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L6",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","styles":null}],null]},[["$","html",null,{"lang":"en","children":["$","body",null,{"children":["$","$L5",null,{"parallelRouterKey":"children","segmentPath":["children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L6",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":[["$","title",null,{"children":"404: This page could not be found."}],["$","div",null,{"style":{"fontFamily":"system-ui,\\"Segoe UI\\",Roboto,Helvetica,Arial,sans-serif,\\"Apple Color Emoji\\",\\"Segoe UI Emoji\\"","height":"100vh","textAlign":"center","display":"flex","flexDirection":"column","alignItems":"center","justifyContent":"center"},"children":["$","div",null,{"children":[["$","style",null,{"dangerouslySetInnerHTML":{"__html":"body{color:#000;background:#fff;margin:0}.next-error-h1{border-right:1px solid rgba(0,0,0,.3)}@media (prefers-color-scheme:dark){body{color:#fff;background:#000}.next-error-h1{border-right:1px solid rgba(255,255,255,.3)}}"}}],["$","h1",null,{"className":"next-error-h1","style":{"display":"inline-block","margin":"0 20px 0 0","padding":"0 23px 0 0","fontSize":24,"fontWeight":500,"verticalAlign":"top","lineHeight":"49px"},"children":"404"}],["$","div",null,{"style":{"display":"inline-block"},"children":["$","h2",null,{"style":{"fontSize":14,"fontWeight":400,"lineHeight":"49px","margin":0},"children":"This page could not be found."}]}]]}]}]],"notFoundStyles":[],"styles":null}]}]}],null],null],"couldBeIntercepted":false,"initialHead":[false,"$L7"],"globalErrorComponent":"$8","missingSlots":"$W9"}]]\n',
    ]);
  </script>
  <script>
    self.__next_f.push([
      1,
      'a:"$Sreact.fragment"\n7:["$","$a","DwzW5r4-q0DsRW72bLFYv",{"children":[["$","meta","0",{"name":"viewport","content":"width=device-width, initial-scale=1"}],["$","meta","1",{"charSet":"utf-8"}]]}]\n2:null\n',
    ]);
  </script>
  <script>
    self.__next_f.push([
      1,
      '4:[["$","h2",null,{"children":"Random Todo"}],["$","ul",null,{"children":[["$","li",null,{"children":["id: ",182]}],["$","li",null,{"children":["todo: ","Explore a nearby trail"]}],["$","li",null,{"children":["completed: ","true"]}],["$","li",null,{"children":["userId: ",188]}]]}]]\n',
    ]);
  </script>
</body>
```

注目すべきは script の`$RC`らへんです。dynamic hole の部分にある template の id が`B:0`、後半送られてきた dynamic rendering の DOM が`S:0`、これらを`$RC("B:0", "S:0")`で置換しているのがわかります。

また、script が直接記述されてることからも前述の通りこれらが**1 つの http リクエスト内で完結**していることもわかります。最初筆者は PPR の仕組みについて、Suspense を利用してるし Render-as-you-fetch してるのかと思ってたのですが、それすらなく 1 つの http レスポンス内でこれらが完結するようになっているのは驚きました。比較実験してないので筆者の理解の範囲における意見ですが、遅延表示するケースにおいてこの実装は理論上非常に高速なのではないかと推測できます。

## SSR/SSG 論争の終焉と PPR 後の議論

PPR 前後で我々 Next.js ユーザーに必要になるパラダイムシフトについても考察したいと思います。前述の例のように、ページの一部を API を介して動的にしたいケースについて考えてみます。PPR 以前なら、このような画面を実装するのに 2 つの選択肢がありました。

- SSG+CSR fetch で一部をクライアントサイドで動的にする
- SSR でページ全体をサーバーサイドで動的にする

これらの選択肢にはそれぞれ懸念が存在します。

SSG+CSR fetchの場合、通信状況によって低速になりうるクライアント〜サーバー間の fetch がレンダリング後に発生するためパフォーマンス的に不利になりえます。また、レンダリング後に fetch する実装や、fetch 先のエンドポイントの実装などが必要になるため実装コストも増えます。

一方ページ全体を SSR するようした場合は実装はシンプルになりえますが、サーバー側で http リクエストのウォーターフォールが発生してしまい、ページ全体のレスポンスが遅くなってしまう可能性が高くなります。場合によってはページのごく一部（前述の例におけるカートなど）の表示のために、ページ全体が遅くなってしまうことも往々にしてあるでしょう。

これら 2 つの選択肢にはトレードオフが付き纏っていたため、SSR にすべきか SSG(+CSR fetch)にすべきか、ケースバイケースなため論争になりがちでした。PPR はシンプルな実装、迅速なレスポンス、1 つの http リクエスト内で完結するという特徴を兼ね備えています。PPR 以降、トレードオフを意識して選択する必要のあったこの SSR/SSG 論争は不要となり、代わりに「**どこまでを static に、どこまでを dynamic にするか**」という議論が重要になってきます。

:::message
現状 App Router は Vercel やセルフホスティングな BFF サーバーを必要とするのが基本系のため、「SSG のみなら BFF サーバーを必要としない」といったメリットについての議論は省略しています。
:::

## PPR の注意点

ここまで PPR を銀の弾丸のように述べてきましたが、PPR にも当然注意点があります。筆者が思う注意点をいくつか紹介したいと思います。

### ページは必ず 200 になる

PPR は画面の静的化された部分については返してしまうため、**ページの http status は必ず 200**になってしまいます

上記の懸念を考慮して SEO に影響のある部分については static rendering してるものとした場合、これが実害になってくるのは監視周りでしょう。そもそも App Router を利用してる場合には Stream によるレスポンスがベースとなるため、http status で監視するだけでは不完全です。なので改めてになりますが、App Router で実装したアプリケーションの監視は http status ではなく個別のエラー発生率などを元に行う必要があるでしょう。

### static rendering で完結できるならその方が良い

PPR はあくまで dynamic rendering を必要とするケースにおける最適化である、ということを念頭におく必要があります。X でやりとりしてて以下のようなコメントをいただきました。

https://twitter.com/sumiren_t/status/1793620259586666643

> 一部を静的化できる＝ SSR でも SG と同じだけ速い、みたいな勘違いがされてないといいなという話でした（自分も前はそういう勘違いをしていたので）

dynamic rendering を含むページでも、PPR なら TTFB(Time to First Bytes)を SSG 相当にすることが可能です。しかし、パフォーマンスにおける「速い」かどうかは、計測する指標によっても異なります。例えば TTI(Time to Interactive)においては、遅延レンダリングの部分が interactive になるまでの時間は SSR とそう変わらないでしょう。

この点は確かに混乱しやすいところかもしれません。PPR におけるパフォーマンスの話をする際には、どの速度指標における話なのか注意して話す必要があるでしょう。

## 感想

PPR は現存する pre-rendering 方式において最も理想的ではないかと筆者は感じています。もちろん裏側では非常に複雑なことをやっているわけですが、我々 Next.js ユーザーからすれば「基本は static rendering、一部を dynamic rendering」というルールに従うのみなのも、シンプルで良い点です。そしてその境界を`<Suspense>`で定義するという使い方も、筆者としてはとても好印象です。

RSC 以降の React では、データフェッチをはじめとしたサーバー側処理もコンポーネント責務にするなど、「必要なことは全てコンポーネントにカプセル化する」という方向性が強まっているように感じます。そしてレンダリングに境界を設け並行性を高めるのが`<Suspense>`です。これらを鑑みても、PPR は非常に昨今の React らしい設計と言えるのではないでしょうか。
