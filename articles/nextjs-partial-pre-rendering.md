---
title: "PPR - pre-rendering新時代の到来とSSR/SSG論争の終焉"
emoji: "👑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: true
---

:::message alert
本稿はNext.js v15.0.0-rc.0時点の情報を元に執筆しており、PPRはさらにexperimentalな機能です。v15.0.0のリリース時や、PPRがstableな機能として提供される際には機能の一部が変更されてる可能性がありますので、ご注意下さい。
:::

**Partial Pre-Rendering**(以降PPR)はNext.js v14.0で発表された、SSRやSSGにならぶ**新たなレンダリングモデル**です。

https://nextjs.org/blog/next-14#partial-prerendering-preview

PPRは前述の通り開発中の機能で、v15のRC版にてexperimentalフラグを有効にすることで利用することができます。`ppr: true`とすれば全部のページが対象となり、`ppr: "incremental"`とすれば`export const experimental_ppr = true`を設定したRouteのみがPPRの対象となります。

https://rc.nextjs.org/docs/app/api-reference/next-config-js/ppr

```js
// next.config.mjs
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    ppr: "incremental", // ppr: boolean | "incremental"
  },
};

export default nextConfig;
```

```tsx
// page.tsx(layout.tsxでも可)
export const experimental_ppr = true;

export default function Page() {
  // ...
}
```

PPRはNext.jsコアチームにとっても重大な機能開発であり、個人的にはとても注目度の高いトピックなのですが、筆者の観測範囲では話題になってるのは一部でそこまで盛り上がってないように感じます。

筆者はPPRによってレンダリングモデルにおける時代がまた1つ新しいものになると考えています。本稿ではPPRとは何か、何を解決しようとしているのか、そしてPPR時代の到来によって何が変わるのか考察したいと思います。

## レンダリングモデルの歴史を振り返る

PPRの話をする前に、これまでのNext.jsのレンダリングモデルについて振り返ってみましょう。これまで、Next.jsがサポートしているレンダリングモデルは3つありました。

- **SSR**: server-side rendering
- **SSG**: static-site generation
- **ISR**: incremental static regeneration

これらがサポートされた歴史的経緯を筆者なりに振り返ってみます。

### Pages Router時代

Next.jsは元々、SSRができるReactフレームワークとして2016/10に登場しました。以下は当時のVercel（旧Zeit）のアナウンス記事です。

https://vercel.com/blog/next

上記v1のアナウンスから長い間Next.jsはSSRをするためのフレームワークでしたが、約3年半後の2019年に登場する[v9.3](https://nextjs.org/blog/next-9-3)でSSG、[v9.5](https://nextjs.org/blog/next-9-5)でISRが導入されたことでNext.jsは複数のレンダリングモデルをサポートするフレームワークとなりました。

当時は[Gatsby](https://www.gatsbyjs.com/)の台頭もありSSG人気が根強く、SSRしかできなかったNext.jsのユーザーはGatsbyに流れることも多かったように思います。npm trendsで確認すると、少々見づらいですが2019年頃はGatsbyの方が上回っています。

_npm trends(Gatsby vs Next.js)_
![npm trends](/images/nextjs-partial-pre-rendering/npm-trends.png)

実際、当時の筆者は好んでGatsbyを使っていました。しかしNext.js v9系で需要の多かったdynamicルーティングやGatsbyが弱かったTypeScript対応などの実装、そして上記SSGやISRのサポートによりNext.jsは一気に注目を集めるようになりました。筆者にはこのv9系で実装された機能群が、今日のNext.jsの人気に繋がったように思えます。

SSRかSSGかというレンダリングモデルの議論は多くのユーザーの関心を集め、これらをどちらもサポートしたNext.jsの選択が、今日の人気を支える重要な要素となっているのです。

### App Router登場以降

上記v9時点では、Next.jsはいわゆるPages Routerしか存在しませんでした。その後[v13](https://nextjs.org/blog/next-13)で発表されたApp Routerでは、RSC(React Server Components)やServer Actions・多層のキャッシュなど多くのパラダイムシフトが必要となりました。App Routerでは、レンダリングモデルに関してはどのような変化があったのでしょうか？

結論から言うとApp Routerは従来同様**SSR/SSG/ISR相当の機能をサポート**していますが、App Routerのドキュメントでは基本的にSSR/SSG/ISRなどの**用語は使用されていません**。

現在App RouterはSSR/SSG/ISRではなく、[static rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default)と[dynamic rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)という2つの概念を使って多くの機能を説明しています。

- **static rendering**: 従来の SSG や ISR 相当で、build 時や revalidate 実行後にレンダリング
  - revalidate なし: SSG 相当
  - revalidate あり: ISR 相当
- **dynamic rendering**: 従来の SSR 相当で、リクエストごとにレンダリング

Pages RouterではSSGかISRかはbuild時に実行する関数で設定する必要があったため静的に決定されていましたが、App Routerにおけるrevalidateは`revalidatePath`や`revalidateTag`で動的に行うことができるので、SSGかISRかは静的に決定されません。そのためNext.jsからするとSSGとISRを区別することに意味がなくなってしまったことが、これらの用語を使わなくなった理由かもしれません。

もう1つ理由として考えられるのは、ISRはVercel以外での運用が難しいと評される機能でネガティブなイメージがついてしまっていたことです。今日では**Cache Handler**によってキャッシュの永続化先を選択できるようになったため、ISR登場時よりはセルフホスティング環境などでも運用しやすくなったはずです。詳しくは筆者の過去記事をご参照ください。

https://zenn.dev/akfm/articles/nextjs-cache-handler-redis

### Streaming SSR

App Router登場時、SSRの中でも技術的な進化がありました。現在App RouterのSSRは**Streaming SSR**をサポートしています。

:::message
Pages Routerについては[v12 のアルファ機能](https://nextjs.org/blog/next-12#react-server-components)として実装されましたが現在は削除され、Streaming SSRをサポートしていません。
:::

Streaming SSRはページのレンダリングの一部を`<Suspense>`で遅延レンダリングにすることが可能で、レンダリングが完了するごとに徐々に結果がクライアントへと送信されます。

```tsx
import { Suspense } from "react";
import { PostFeed, Weather } from "./Components";

export default function Posts() {
  return (
    <section>
      <Suspense fallback={<p>Loading feed...</p>}>
        <PostFeed />
      </Suspense>
      <Suspense fallback={<p>Loading weather...</p>}>
        <Weather />
      </Suspense>
    </section>
  );
}
```

上記の実装例においては、最初に`fallback`(`Loading feed...`や`Loading weather...`)が表示され、サーバー側で`<PostFeed>`や`<Weather>`のレンダリングが完了すると順々にクライアントにレンダリング結果が送信されて`fallback`が置き換えられます。また、これらが**1つのHTTPレスポンスで完結**し、レスポンスのHTMLに`<PostFeed>`や`<Weather>`のHTMLが含まれるのでSEO観点もフォローしていることが大きな特徴です。

より詳細にStreaming SSRの仕組みが知りたい方は、uhyoさんの記事が参考になると思います。

https://zenn.dev/uhyo/books/rsc-without-nextjs/viewer/streaming-ssr

## SSG/SSRにおける静的・動的データの混在

ページを構成するのに必要なデータが、静的なデータ(キャッシュ可能)と動的データ(キャッシュ不可能)で混在することがあります。例えばECサイトにおいて、商品情報自体はbuild時やrevalidateごとに取得・キャッシュできる場合もありますが、ログイン情報はキャッシュできないので動的に取得する必要があります。

このように静的データと動的データが混在する場合、App Routerでは大きく以下2つの実装パターンがありました。

- **SSG+Client fetch**: ページ自体はSSGにしてクライアントサイドで動的データをfetchする
- **Streaming SSR**: 静的データはキャッシュ([Data Cache](https://nextjs.org/docs/app/building-your-application/caching))を利用して高速化しつつ、ページの一部を`<Suspense>`で遅延レンダリングにする

しかしこれらは互いにいくつかの点でメリット・デメリットが相反する関係にあり、状況によって最適解が異なります。そのため、これらの選択の議論や説明時にはSSGやStreaming SSRについての高度な理解が必要になります。

これらのメリデメについて簡単に整理してみます。なおTTFBはTime to First Bytesの略です。

| 観点                 | SSG+Client fetch | Streaming SSR |
| -------------------- | ---------------- | ------------- |
| TTFB                 | 有利             | 若干不利      |
| HTTPラウンドトリップ | 複数回           | 1回           |
| CDNキャッシュ        | 可能             | 不可          |
| 実装                 | 冗長になりがち   | シンプル      |

:::message
App RouterはVercelやセルフホスティングサーバーを用意することが最も基本的な運用パターンとなっているので、「サーバーが必要・不要」と言った観点は省略しています。
:::

通常のSSRに比べてStreaming SSRならTTFBは改善しますが、それでもレンダリング時に静的なファイルを返すだけのSSGの方が当然有利です。

一方実装観点ではClient fetchの場合クライアントサイド処理とサーバー側のエンドポイントを繋ぐ処理(API Routes、tRPC、GraphQLなど)が必要になるので、Streaming SSRの方がシンプルと筆者は考えます。また、Streaming SSRではHTTPのラウンドトリップが1回で済む分動的要素が表示されるまでの時間は短くなると考えられる点も、パフォーマンス観点から評価できます。

このようにStreaming SSRは多くのメリットを持っている一方、SSGが持つTTFBの速度は得られないことがトレードオフでした。これを解消する手段として登場したのが、本稿の主題である**PPR**です。

## PPRとは

PPRはStreaming SSRをさらに進化させた技術で、**ページをstatic renderingとしつつ、部分的にdynamic renderingにする**ことが可能なレンダリングモデルです。SSG・ISRのページの一部にSSRな部分を組み合わせられるようなイメージ、あるいはStreaming SSRのスケルトン部分をSSG/ISRにするイメージです。[公式の説明](https://rc.nextjs.org/learn/dashboard-app/partial-prerendering#what-is-partial-prerendering)よりECサイトの商品ページの例を拝借すると、以下のような構成が可能になります。

![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

商品ページ全体やナビゲーションはstatic renderingで静的化され、一方カートやレコメンド情報といったユーザーごとに異なるUI部分はdynamic renderingにすることができます。もちろん商品情報自体が更新されることもあるでしょうが、この例では必要に応じてrevalidateすることを想定しています。

### 静的化とStreamingレンダリングの恩恵

Streaming SSRでは`<Suspense>`の外側について、リクエストごとに以下のような処理がされてました。

1. Server Components(多段階計算の1段目)を実行
2. Client Components(多段階計算の2段目)を実行
3. 1と2の結果(Reactツリー)からHTMLを生成
4. 3の結果をレスポンスに流す

Componentsの実行を多段計算と称してることについてはuhyoさんの記事をご参照ください。

https://zenn.dev/uhyo/articles/react-server-components-multi-stage#%E4%B8%80%E8%A8%80%E3%81%A7react-server-components%E3%82%92%E7%90%86%E8%A7%A3%E3%81%99%E3%82%8B

PPRではこの1~3をbuild時に実行し静的化するため、Next.jsサーバーは初期表示に使うHTMLの送信を**より高速なレスポンス**にすることができます。

### PPRの挙動観察

PPRにおいてdynamic renderingされる部分を遅延させる場合、それらは**dynamic hole**、もしくはasync holeやただのholeと呼ばれます。PPRを有効化して実際にdynamic holeが置き換わる様子を観察してみましょう。

以下のサンプルコードを元に挙動を観察してみます。

```tsx
// app/ppr/page.tsx
import { Suspense } from "react";
import { setTimeout } from "node:timers/promises";

// 📍PPRを有効化
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

`<RandomTodo>`はリクエストの度にランダムなTODO情報を取得するコンポーネントです。
ページ自体である`<Home>`はstatic renderingですが、APIへのfetchに`no-store`を指定しているので、`<RandomTodo>`はdynamic renderingとなります。また、今回はStreamの様子を観察したいのでリクエスト後にあえて3秒遅延させています。

:::message
今回の主題ではないのですが、v15.0.0-rc.0時点ではデフォルトでfetchは`no-store`ですが、**明示的に指定しないとdynamic renderingにならない**という仕様になっているのでサンプルコードでは明示的に指定しています。同様の対策として[`unstable_noStore`](https://nextjs.org/docs/app/api-reference/functions/unstable_noStore)などの[dynamic functions](https://nextjs.org/docs/app/building-your-application/routing/route-handlers#dynamic-functions)を使ってdynamic renderingにすることも可能です。

デフォルトの仕様については[RC中に変更される可能性](https://x.com/feedthejim/status/1794778189354705190)が示唆されています。
:::

実際に画面を表示した時の様子が以下です。

_初期表示_
![stream start](/images/nextjs-partial-pre-rendering/ppr-stream-start.png)

_約 3 秒後_
![stream end](/images/nextjs-partial-pre-rendering/ppr-stream-end.png)

初期表示の時点では`<Suspense>`の`fallback`に指定した`loading...`が表示されており、その後`<RandomTodo>`のレンダリング結果が送信されてくると`loading...`が置き換えられている様子が確認できます。

DevToolsを確認すると、レスポンスも初期表示用のHTMLが送信された時点で一度止まっていることがわかります。初期表示時点で送信されてきた`<body>`配下のHTMLは以下です。

```html
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
```

Streaming SSRだと`<Home>`はリクエストの度に毎回計算する必要がありましたが、PPRでは静的化されているので`<Home>`がレンダリングされるのはbuild時やrevalidate後のみで、リクエストの度に計算されるのは`<RandomTodo>`のみとなります。そのためNext.jsサーバーは上記のHTMLを含む静的ファイルを即座にクライアントへ送信することができます。

dynamic renderingな`<RandomTodo>`以降のHTMLはStreaming SSR同様、レンダリングが完了次第送信されます。

```html
<div hidden id="S:0">
  <h2>Random Todo</h2>
  <ul>
    <li>
      id:
      <!-- -->
      253
    </li>
    <li>
      todo:
      <!-- -->
      Try a new fitness class like aerial yoga or barre
    </li>
    <li>
      completed:
      <!-- -->
      true
    </li>
    <li>
      userId:
      <!-- -->
      21
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
    '1:I[4129,[],""]\n3:"$Sreact.suspense"\n5:I[8330,[],""]\n6:I[3533,[],""]\n8:I[6344,[],""]\n9:[]\n',
  ]);
</script>
<script>
  self.__next_f.push([
    1,
    '0:[null,["$","$L1",null,{"buildId":"u-TCHmQLHODl6ILIXZKdy","assetPrefix":"","initialCanonicalUrl":"/ppr","initialTree":["",{"children":["ppr",{"children":["__PAGE__",{}]}]},"$undefined","$undefined",true],"initialSeedData":["",{"children":["ppr",{"children":["__PAGE__",{},[["$L2",["$","main",null,{"children":[["$","h1",null,{"children":"PPR Page"}],["$","$3",null,{"fallback":"loading...","children":"$L4"}]]}]],null],null]},["$","$L5",null,{"parallelRouterKey":"children","segmentPath":["children","ppr","children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L6",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","styles":null}],null]},[["$","html",null,{"lang":"en","children":["$","body",null,{"children":["$","$L5",null,{"parallelRouterKey":"children","segmentPath":["children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L6",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":[["$","title",null,{"children":"404: This page could not be found."}],["$","div",null,{"style":{"fontFamily":"system-ui,\\"Segoe UI\\",Roboto,Helvetica,Arial,sans-serif,\\"Apple Color Emoji\\",\\"Segoe UI Emoji\\"","height":"100vh","textAlign":"center","display":"flex","flexDirection":"column","alignItems":"center","justifyContent":"center"},"children":["$","div",null,{"children":[["$","style",null,{"dangerouslySetInnerHTML":{"__html":"body{color:#000;background:#fff;margin:0}.next-error-h1{border-right:1px solid rgba(0,0,0,.3)}@media (prefers-color-scheme:dark){body{color:#fff;background:#000}.next-error-h1{border-right:1px solid rgba(255,255,255,.3)}}"}}],["$","h1",null,{"className":"next-error-h1","style":{"display":"inline-block","margin":"0 20px 0 0","padding":"0 23px 0 0","fontSize":24,"fontWeight":500,"verticalAlign":"top","lineHeight":"49px"},"children":"404"}],["$","div",null,{"style":{"display":"inline-block"},"children":["$","h2",null,{"style":{"fontSize":14,"fontWeight":400,"lineHeight":"49px","margin":0},"children":"This page could not be found."}]}]]}]}]],"notFoundStyles":[],"styles":null}]}]}],null],null],"couldBeIntercepted":false,"initialHead":[false,"$L7"],"globalErrorComponent":"$8","missingSlots":"$W9"}]]\n',
  ]);
</script>
<script>
  self.__next_f.push([
    1,
    'a:"$Sreact.fragment"\n7:["$","$a","yuyzwuCpBYflRRVYLHWqg",{"children":[["$","meta","0",{"name":"viewport","content":"width=device-width, initial-scale=1"}],["$","meta","1",{"charSet":"utf-8"}],["$","title","2",{"children":"Create Next App"}],["$","meta","3",{"name":"description","content":"Generated by create next app"}],["$","link","4",{"rel":"icon","href":"/favicon.ico","type":"image/x-icon","sizes":"16x16"}]]}]\n2:null\n',
  ]);
</script>
<script>
  self.__next_f.push([
    1,
    '4:[["$","h2",null,{"children":"Random Todo"}],["$","ul",null,{"children":[["$","li",null,{"children":["id: ",253]}],["$","li",null,{"children":["todo: ","Try a new fitness class like aerial yoga or barre"]}],["$","li",null,{"children":["completed: ","true"]}],["$","li",null,{"children":["userId: ",21]}]]}]]\n',
  ]);
</script>
```

注目すべきは`<script>`の`$RC`周辺です。最初に送られてきたHTMLにある`<template>`のidが`B:0`、後半送られてきた`<RandomTodo>`のHTMLが`S:0`、これらを`$RC("B:0", "S:0")`で置換しているのがわかります。また、`<script>`が直接記述されてることからも前述の通りこれらが**1つのHTTPレスポンスで完結**していることもわかります。

## PPR考察

PPRの動作についてはおおよそ理解いただけたかと思いますが、実際我々はこのPPRをどう受け止めるべきなのでしょう？機能を知ることと役割を知ることは別な議論です。筆者なりにPPRがもたらす変化について、いくつか考察してみたいと思います。

### SSG+Client fetch/Streaming SSRとの比較

[SSG/SSRにおける静的・動的データの混在](#ssgssrにおける静的動的データの混在)で示した表に、PPRを追加して比較してみます。

| 観点                 | PPR          | SSG+Client fetch | Streaming SSR |
| -------------------- | ------------ | ---------------- | ------------- |
| TTFB                 | **有利**     | 有利             | 若干不利      |
| HTTPラウンドトリップ | **1回**      | 複数回           | 1回           |
| CDNキャッシュ        | **不可**     | 可能             | 不可          |
| 実装                 | **シンプル** | 冗長になりがち   | シンプル      |

PPRではSSG+Client fetch相当のTTFBと実装のシンプルさを同時に得られます。HTML内に動的な要素が含まれるためCDNキャッシュこそできませんが、他の点においてはSSG+Client fetchとStreaming SSR両方のメリットを併せ持っています。

### PPRによるReactらしい設計責務

RSC以降のReactでは、データフェッチをはじめとしたサーバー側処理もコンポーネント責務にするなど、「必要なことは全てコンポーネントにカプセル化する」という方向性が強まっているように感じます。そしてレンダリングに境界を設け並行性を高めるのが`<Suspense>`です。

そのため、`<Supense>`境界をもってdynamic renderingとstatic renderingが切り替えることが可能になるPPRは、非常に昨今の**Reactらしい設計**ではないかと筆者は考えています。実際、PPRを利用するには冒頭述べたexperimental設定を除き**新たなAPIを学習する必要がない**こともReactらしい設計に則ってることの裏付けと言えるでしょう。

### SSR/SSG論争の終焉

昨今のNext.jsも「必要なことは全てコンポーネントにカプセル化する」という方向性により、ページ単位で考えることが減ってきている傾向がみられます。もちろんWebの仕組み上URLを元にしているので、ページ単位で考えなければいけないmeta情報やURLに含まれる動的パスやパラメータなど、「ページ」という概念を無くすことはできません。

しかし、ことレンダリングモデルについては必ずしもページという概念が必須ではありません。従来はSSR/SSG/ISRどれをとってもページ単位で考える必要がありましたが、PPR以降はより細粒度な`<Suspense>`境界を元にしたUI単位で考えることが可能になります。これにより「SSRにすべきかSSGにすべきか」といった論争は過去のものとなり、PPR以降はより細粒度な「**どこまでをstaticに、どこからをdynamicにするか**」という議論へとシフトすることになります。

## PPRのデメリット考察

ここまでPPRを銀の弾丸のように述べてきましたが、PPRにも当然注意点があります。筆者が思う注意点をいくつか紹介したいと思います。

### ページは必ず200になる

PPRは画面の静的化された部分については返してしまうため、**ページのHTTP Statusは必ず200**になってしまいます

これが実害になってくるのは監視周りでしょう。そもそもApp Routerを利用してる場合にはStreamによるレスポンスがベースとなるため、HTTP Statusで監視するだけでは不完全です。なのでPPRに限らない話ではありますが、App Routerで実装したアプリケーションの監視はHTTP Statusではなく個別のエラー発生率などを元に行う必要があるでしょう。

### static renderingで完結できるならその方が良い

PPRはあくまでdynamic renderingを必要とするケースにおける最適化である、ということを念頭におく必要があります。Xでやりとりしてて以下のようなコメントをいただきました。

https://twitter.com/sumiren_t/status/1793620259586666643

> 一部を静的化できる＝SSRでもSGと同じだけ速い、みたいな勘違いがされてないといいなという話でした（自分も前はそういう勘違いをしていたので）

確かに、dynamic renderingを含むページでもPPRならTTFBをSSGに近づけることが可能です。しかし、パフォーマンスというのはTTFBだけで測るものではありません。例えばTime to InteractiveにおいてはPPRでもSSRでも大きくは変わらないため、ページ全体をSSGにできるならその方が有利でしょう。

PPRはパフォーマンスにおける銀の弾丸ではないものの一部パフォーマンスが改善されることは確かで、混乱しやすいところかもしれません。PPRにおいても、パフォーマンスの話はどの速度指標についての話なのか注意する必要があるでしょう。

## 感想

PPRは現存するレンダリングモデルにおいて最も理想的ではないかと筆者は感じています。もちろん裏側では非常に複雑なことをやっているわけですが、我々Next.jsユーザーからすれば「基本はstatic rendering、一部をdynamic rendering」というルールに従うのみなのも、シンプルで良い点です。そしてその境界を`<Suspense>`で定義するという使い方も、筆者としてはとても好印象です。

v15のGA時に[PPRのロードマップ](https://nextjs.org/blog/next-15-rc#incremental-adoption-of-partial-prerendering-experimental)が発表されるとのことです。PPRが今後どういった進化を遂げるのか、非常に楽しみです。
