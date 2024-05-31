---
title: "Partial Pre-Rendering: 新時代レンダリングモデルとStreamingレンダリングの威力"
emoji: "👑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: true
---

:::message alert
本稿は Next.js v15.0.0-rc.0 時点の情報を元に執筆しており、PPR はさらに experimental な機能です。v15.0.0 のリリース時や、PPR が stable な機能として提供される際には機能の一部が変更されてる可能性がありますので、ご注意下さい。
:::

**Partial Pre-Rendering**(以降 PPR)は Next.js v14.0 で発表された、SSR や SSG にならぶ**新たなレンダリングモデル**です。

https://nextjs.org/blog/next-14#partial-prerendering-preview

PPR は前述の通り開発中の機能で、v15 の RC 版にて experimental フラグを有効にすることで利用することができます。`ppr: true`とすれば全部のページが対象となり、`ppr: "incremental"`とすれば`export const experimental_ppr = true`を設定した Route Segment のみが PPR の対象となります。

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

筆者は PPR によってレンダリングモデルにおける時代がまた 1 つ新しいものになると考えています。本稿では PPR とは何か、何を解決しようとしているのか、そして PPR 時代の到来によって何が変わるのか考察したいと思います。

## レンダリングモデルの歴史を振り返る

PPR の話をする前に、これまでの Next.js のレンダリングモデルについて振り返ってみましょう。これまで、Next.js がサポートしているレンダリングモデルは 3 つありました。

- **SSR**: server-side rendering
- **SSG**: static-site generation
- **ISR**: incremental static regeneration

これらがサポートされた歴史的経緯を筆者なりに振り返ってみます。

### Pages Router 時代

Next.js は元々、SSR ができる React フレームワークとして 2016/10 に登場しました。以下は当時の Vercel（旧 Zeit）のアナウンス記事です。

https://vercel.com/blog/next

上記 v1 のアナウンスから長い間 Next.js は SSR をするためのフレームワークでしたが、約 3 年半後の 2019 年に登場する[v9.3](https://nextjs.org/blog/next-9-3)で SSG、[v9.5](https://nextjs.org/blog/next-9-5)で ISR が導入されたことで Next.js は複数のレンダリングモデルをサポートするフレームワークとなりました。

当時は[Gatsby](https://www.gatsbyjs.com/)の台頭もあり SSG 人気が根強く、SSR しかできなかった Next.js のユーザーは Gatsby に流れることも多かったように思います。実際 npm trends で確認すると 2019 年頃は Gatsby の方が上回っています(見づらくてすいません)。

_npm trends(Gatsby vs Next.js)_
![npm trends](/images/nextjs-partial-pre-rendering/npm-trends.png)

実際、当時の筆者は好んで Gatsby を使っていました。しかし Next.js v9 系で需要の多かった dynamic ルーティングや Gatsby が弱かった TypeScript 対応などの実装、そして上記 SSG や ISR のサポートにより Next.js は一気に注目を集めるようになりました。筆者にはこの v9 系で実装された機能群が、 今日の Next.js の人気に繋がったように思えます。

SSR か SSG かというレンダリングモデルの議論は多くのユーザーの関心を集め、これらをどちらもサポートした Next.js の選択が、今日の人気を支える重要な要素となっているのです。

### App Router 登場以降

上記 v9 時点では、Next.js はいわゆる Pages Router しか存在しませんでした。その後[v13](https://nextjs.org/blog/next-13)で発表された App Router では、RSC(React Server Components)や Server Actions・多層のキャッシュなど多くのパラダイムシフトが必要となりました。App Router では、レンダリングモデルに関してはどのような変化があったのでしょうか？

結論から言うと App Router は従来同様**SSR/SSG/ISR 相当の機能をサポート**していますが、App Router のドキュメントでは基本的に SSR/SSG/ISR などの**用語は使用されていません**。

現在 App Router は SSR/SSG/ISR ではなく、[static rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default)と[dynamic rendering](https://rc.nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)という 2 つの概念を使って多くの機能を説明しています。

- **static rendering**: 従来の SSG や ISR 相当で、build 時や revalidate 実行後にレンダリング
  - revalidate なし: SSG 相当
  - revalidate あり: ISR 相当
- **dynamic rendering**: 従来の SSR 相当で、リクエストごとにレンダリング

Pages Router では SSG か ISR かは build 時に実行する関数で設定する必要があったため静的に決定されていましたが、App Router における revalidate は`revalidatePath`や`revalidateTag`で動的に行うことができるので、SSG か ISR かは静的に決定されません。そのため Next.js からすると SSG と ISR を区別することに意味がなくなってしまったことが、これらの用語を使わなくなった理由かもしれません。

もう 1 つ理由として考えられるのは、ISR は Vercel 以外での運用が難しいと評される機能でネガティブなイメージがついてしまっていたことです。今日では**Cache Handler**によってキャッシュの永続化先を選択できるようになったため、ISR 登場時よりはセルフホスティング環境などでも運用しやすくなったはずです。詳しくは筆者の過去記事をご参照ください。

https://zenn.dev/akfm/articles/nextjs-cache-handler-redis

## Streaming SSR

ここまで SSR と一言に括ってしまってましたが、SSR の中でも技術的な進化がありました。現在 App Router の SSR は**Streaming SSR**をサポートしています。

:::message
Pages Router については[v12 のアルファ機能](https://nextjs.org/blog/next-12#react-server-components)として実装されましたが現在は削除され、Streaming SSR をサポートしていません。
:::

Streaming SSR はページのレンダリングの一部を`<Suspense>`で遅延レンダリングにすることが可能で、レンダリングが完了するごとに徐々に結果がクライアントへと送信されます。以下の実装例で考えてみます。

```tsx
// app/streaming-ssr/page.tsx
import { Suspense } from "react";
import { setTimeout } from "node:timers/promises";

// 📍PPRは無効化
// export const experimental_ppr = true;

export default function Home() {
  return (
    <main>
      <h1>Streaming SSR Page</h1>
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

`<RandomTodo>`はリクエストの度にランダムな TODO 情報を取得するコンポーネントです。API への fetch に`no-store`を指定しているので、ページ全体が dynamic rendering となります。また、今回は Stream の様子を観察したいのでリクエスト後にあえて 3 秒遅延させています。

:::message
今回の主題ではないのですが、v15.0.0-rc.0 時点ではデフォルトで fetch は`no-store`ですが、**明示的に指定しないと dynamic rendering にならない**という仕様になっているのでサンプルコードでは明示的に指定しています。同様の対策として[`unstable_noStore`](https://nextjs.org/docs/app/api-reference/functions/unstable_noStore)などの[dynamic functions](https://nextjs.org/docs/app/building-your-application/routing/route-handlers#dynamic-functions)を使って dynamic rendering にすることも可能です。

デフォルトの仕様については[RC 中に変更される可能性](https://x.com/feedthejim/status/1794778189354705190)が示唆されています。
:::

実際に画面を表示した時の様子が以下です。

_初期表示_
![stream start](/images/nextjs-partial-pre-rendering/stream-start.png)

_約 3 秒後_
![stream end](/images/nextjs-partial-pre-rendering/stream-end.png)

初期表示の時点では`<Suspense>`の`fallback`に指定した`loading...`が表示されており、その後`<RandomTodo>`のレンダリング結果が送信されてくると`loading...`が置き換えられている様子が確認できます。

DevToolsを確認すると、レスポンスのhtmlも初期表示用のDOMが送信された時点で一度止まっていることがわかります。初期表示時点で送信されてきた`<body>`配下のDOMは以下です。

```html
<main>
  <h1>Streaming SSR Page</h1>
  <!--$?-->
  <template id="B:0"></template>
  loading...
  <!--/$-->
</main>
<script
  src="/_next/static/chunks/webpack-b5d81ab04c5b38dd.js"
  async=""
></script>
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
    '0:[null,["$","$L1",null,{"buildId":"IVswg2uNFzZunMK3bKt42","assetPrefix":"","initialCanonicalUrl":"/streaming-ssr","initialTree":["",{"children":["streaming-ssr",{"children":["__PAGE__",{}]}]},"$undefined","$undefined",true],"initialSeedData":["",{"children":["streaming-ssr",{"children":["__PAGE__",{},[["$L2",["$","main",null,{"children":[["$","h1",null,{"children":"Streaming SSR Page"}],["$","$3",null,{"fallback":"loading...","children":"$L4"}]]}]],null],null]},["$","$L5",null,{"parallelRouterKey":"children","segmentPath":["children","streaming-ssr","children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L6",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","styles":null}],null]},[["$","html",null,{"lang":"en","children":["$","body",null,{"children":["$","$L5",null,{"parallelRouterKey":"children","segmentPath":["children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L6",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":[["$","title",null,{"children":"404: This page could not be found."}],["$","div",null,{"style":{"fontFamily":"system-ui,\\"Segoe UI\\",Roboto,Helvetica,Arial,sans-serif,\\"Apple Color Emoji\\",\\"Segoe UI Emoji\\"","height":"100vh","textAlign":"center","display":"flex","flexDirection":"column","alignItems":"center","justifyContent":"center"},"children":["$","div",null,{"children":[["$","style",null,{"dangerouslySetInnerHTML":{"__html":"body{color:#000;background:#fff;margin:0}.next-error-h1{border-right:1px solid rgba(0,0,0,.3)}@media (prefers-color-scheme:dark){body{color:#fff;background:#000}.next-error-h1{border-right:1px solid rgba(255,255,255,.3)}}"}}],["$","h1",null,{"className":"next-error-h1","style":{"display":"inline-block","margin":"0 20px 0 0","padding":"0 23px 0 0","fontSize":24,"fontWeight":500,"verticalAlign":"top","lineHeight":"49px"},"children":"404"}],["$","div",null,{"style":{"display":"inline-block"},"children":["$","h2",null,{"style":{"fontSize":14,"fontWeight":400,"lineHeight":"49px","margin":0},"children":"This page could not be found."}]}]]}]}]],"notFoundStyles":[],"styles":null}]}]}],null],null],"couldBeIntercepted":false,"initialHead":[false,"$L7"],"globalErrorComponent":"$8","missingSlots":"$W9"}]]\n',
  ]);
</script>
<script>
  self.__next_f.push([
    1,
    'a:"$Sreact.fragment"\n7:["$","$a","YhMIPzqOngOVCvTPC0l1l",{"children":[["$","meta","0",{"name":"viewport","content":"width=device-width, initial-scale=1"}],["$","meta","1",{"charSet":"utf-8"}],["$","title","2",{"children":"Create Next App"}],["$","meta","3",{"name":"description","content":"Generated by create next app"}],["$","link","4",{"rel":"icon","href":"/favicon.ico","type":"image/x-icon","sizes":"16x16"}]]}]\n2:null\n',
  ]);
</script>
```

約 3 秒後、`<RandomTodo>`のレンダリングが完了した時点で残りのDOM として以下が送信されてきます。

```html
<script>
  self.__next_f.push([
    1,
    '4:[["$","h2",null,{"children":"Random Todo"}],["$","ul",null,{"children":[["$","li",null,{"children":["id: ",133]}],["$","li",null,{"children":["todo: ","Learn GraphQL"]}],["$","li",null,{"children":["completed: ","true"]}],["$","li",null,{"children":["userId: ",169]}]]}]]\n',
  ]);
</script>
<div hidden id="S:0">
  <h2>Random Todo</h2>
  <ul>
    <li>
      id:
      <!-- -->
      133
    </li>
    <li>
      todo:
      <!-- -->
      Learn GraphQL
    </li>
    <li>
      completed:
      <!-- -->
      true
    </li>
    <li>
      userId:
      <!-- -->
      169
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
```

注目すべきは script の`$RC`らへんです。最初に送られてきたDOMにある `<template>` の id が`B:0`、後半送られてきた `<RandomTodo>` の DOM が`S:0`、これらを`$RC("B:0", "S:0")`で置換しているのがわかります。また、script が直接記述されてることからも前述の通りこれらが**1 つの http レスポンスで完結**していることもわかります。

より詳細にStreaming SSR の仕組みが知りたい方は、uhyo さんの記事が参考になると思います。

https://zenn.dev/uhyo/books/rsc-without-nextjs/viewer/streaming-ssr

### SSG + CSR fetch vs Streaming SSR

上述の例のようなページの一部を動的にしたい場合、ページ自体はSSGにしておいて動的な部分だけをCSR fetchで動的にするという手段もあります。そうすればNext.jsサーバーからしたらページ自体は静的なので、高速で安定したレスポンスも実現することができます。

しかしSSG+CSR fetchの場合、パフォーマンス的には上記の理由からTTFB(Time To First Byte)は短縮が見込めますが、TTI(Time To Interactive)はhttp通信のラウンドトリップが追加で発生するなどの理由から遅くなる可能性があります。Streaming SSRはTTFBでは不利ですがTTI観点では1 http レスポンスで完結するので有利になります。

## PPR とは

PPR は Streaming SSR をさらに進化させた技術で、**ページを static rendering しつつ、部分的に dynamic rendering にする**ことが可能なレンダリングモデルです。SSG・ISR のページの一部に SSR な部分を組み合わせられるようなイメージ、あるいは Streaming SSR のスケルトン部分を SSG/ISR にするイメージです。[公式の説明](https://rc.nextjs.org/learn/dashboard-app/partial-prerendering#what-is-partial-prerendering)より EC サイトの商品ページの例を拝借すると、以下のような構成が可能になります。

![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

商品ページ全体やナビゲーションは static rendering で静的化され、一方カートやレコメンド情報といったユーザーごとに異なる UI 部分は dynamic rendering にすることができます。もちろん商品情報自体が更新されることもあるでしょうが、この例では必要に応じて revalidate することを想定しています。

### PPRによるメリット

Streaming SSRでは`<Suspense>`の外側について、リクエストごとに以下のような処理がされてました。

1. Server Componentsを実行 
2. Client Componentsを実行 
3. 1と2の結果からHTMLを生成 
4. 3の結果をレスポンスに流す

PPRではこの1~3をbuild時に実行し静的化するため、Next.jsサーバーは初期表示に使うDOMの送信を**より高速で安定したレスポンス**にすることができます。[SSG + CSR fetch vs Streaming SSR](#ssg--csr-fetch-vs-streaming-ssr)のトレードオフの議論において、いいとこどりをしたのがPPRとも言えます。

前述の例で言うと、`<Home>`はリクエストの度に毎回計算する必要はありませんでしたが、`<RandomTodo>`によってページ全体がdynamic renderingだったので毎回上記の処理を実行していました。PPRが有効になると、リクエストの度に計算されるのは`<RandomTodo>`のみとなります。

また、PPRはStreaming SSRを進化させたものなので、冒頭述べたexperimental設定を除き**新たなAPIを学習する必要はありません**。我々開発者はStreaming SSR同様`<Suspense>`で遅延レンダリングする部分を指定するだけで、PPRを実現できます。

### PPR の観察

PPR において遅延レンダリングさせる部分が dynamic rendering な場合、それらは**dynamic hole**、もしくは async hole やただの hole と呼ばれます。

PPR によって dynamic hole が置き換わる様子も観察してみましょう。Streaming SSR で使ったサンプルコードを元に`/app/ppr/page.tsx`を作成し、挙動を観察してみます。

```tsx
// app/ppr/page.tsx

// ...

// 📍PPRを有効化
export const experimental_ppr = true;

// 📍h1を修正したのみ
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
  // ...
}

// ...
```

PPR が有効化されたので Streaming SSR と違い、ページ自体である`<Home>`は static rendering になり`<RandomTodo>`の部分のみが dynamic rendering になります。

このページの表示とレスポンスの様子は以下のようになります。

_初期表示_
![stream start](/images/nextjs-partial-pre-rendering/ppr-stream-start.png)

_約 3 秒後_
![stream end](/images/nextjs-partial-pre-rendering/ppr-stream-end.png)

基本的な動作は Streaming SSR 同様で、初期表示時点では`loading...`が表示されています。dynamic rendering が完了するとクライアントに残りの DOM が送信され、`loading...`が`Random Todo`に置き換えられます。

レンダリング結果の DOM も確認してみます。最初にクライアントへ送信される DOM は以下です。

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

Streaming SSRの場合、上記のような最初に送信されるDOMもサーバー側でSSRされたものでしたが、PPRではbuild時やrevalidate後のリクエストでこのDOMを生成するのでNext.jsサーバーは静的ファイルをクライアントへ即座に送信するのみです。

以降のDOMはStreaming SSR 同様、レンダリングが完了次第送信されます。

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

多少scriptタグの位置が異なれど、基本的にはStreaming SSRと同じようなレスポンスが得られました。

比較実験してないので筆者の理解の範囲における意見ですが、遅延表示するケースにおいてこの実装は理論上非常に高速なのではないかと推測できます。

## PPR へのパラダイムシフト

PPR 前後で我々 Next.js ユーザーに必要になるパラダイムシフトについても考察したいと思います。前述の例のように、ページの一部を API を介して動的にしたいケースについて考えてみます。PPR 以前なら、このような画面を実装するのに 3 つの選択肢がありました。

- SSG+CSR fetch で一部をクライアントサイドで動的にする
- ページ全体を SSR する
- Streaming SSR を利用してページの一部を遅延レンダリングする

SSG+CSR fetch の場合、通信状況によって低速になりうるクライアント〜サーバー間の fetch がレンダリング後に発生するためパフォーマンス的に不利になりえます。また、レンダリング後に fetch する実装や、fetch 先のエンドポイントの実装などが必要になるため実装コストも増えます。

ページ全体を SSR するようした場合は実装はシンプルになりえますが、サーバー側で複数 fetch が発生すると http リクエストのウォーターフォールが発生してしまい、ボトルネックに引っ張られてページ全体のレスポンスが遅くなってしまう可能性があります。

Streaming SSR はページ全体を SSR する際に発生した課題を解決しますが、前述の例の商品ページのようなケースにおいて高速なレスポンスを目指す場合、商品ページへの fetch をキャッシュする必要がありました。そのために App Router では Data Cache をデフォルトで強くキャッシュする戦略をとっていたと考えられます。

Streaming SSR をさらに発展させ、Data Cache ではなく static rendering にすることでより高速でシンプルな設計になったのが PPR です。PPR によってこの SSR/SSG 論争や Data Cache の管理は減り、代わりにこれからは「**どこまでを static に、どこまでを dynamic にするか**」という議論と選択が重要になってきます。

:::message
現状 App Router は Vercel やセルフホスティングな BFF サーバーを必要とするのが基本系のため、「SSG のみなら BFF サーバーを必要としない」といったメリットについての議論は省略しています。
:::

## PPR の注意点

ここまで PPR を銀の弾丸のように述べてきましたが、PPR にも当然注意点があります。筆者が思う注意点をいくつか紹介したいと思います。

### ページは必ず 200 になる

PPR は画面の静的化された部分については返してしまうため、**ページの http status は必ず 200**になってしまいます

これが実害になってくるのは監視周りでしょう。そもそも App Router を利用してる場合には Stream によるレスポンスがベースとなるため、http status で監視するだけでは不完全です。なので PPR に限らない話ではありますが、App Router で実装したアプリケーションの監視は http status ではなく個別のエラー発生率などを元に行う必要があるでしょう。

### static rendering で完結できるならその方が良い

PPR はあくまで dynamic rendering を必要とするケースにおける最適化である、ということを念頭におく必要があります。X でやりとりしてて以下のようなコメントをいただきました。

https://twitter.com/sumiren_t/status/1793620259586666643

> 一部を静的化できる＝ SSR でも SG と同じだけ速い、みたいな勘違いがされてないといいなという話でした（自分も前はそういう勘違いをしていたので）

dynamic rendering を含むページでも、PPR なら TTFB(Time to First Bytes)を SSG 相当にすることが可能です。しかし、パフォーマンスにおける「速い」かどうかは、計測する指標によっても異なります。例えば TTI(Time to Interactive)においては、遅延レンダリングの部分が interactive になるまでの時間は SSR とそう変わらないでしょうし、SSG+CSR fetch であれば前述の理由から TTFB は早くても TTI については SSR 以下になってしまう可能性も十分あります。

パフォーマンス観点は確かに混乱しやすいところかもしれません。PPR においても、パフォーマンスの話はどの速度指標についての話なのか注意する必要があるでしょう。

## 感想

PPR は現存するレンダリングモデルにおいて最も理想的ではないかと筆者は感じています。もちろん裏側では非常に複雑なことをやっているわけですが、我々 Next.js ユーザーからすれば「基本は static rendering、一部を dynamic rendering」というルールに従うのみなのも、シンプルで良い点です。そしてその境界を`<Suspense>`で定義するという使い方も、筆者としてはとても好印象です。

RSC 以降の React では、データフェッチをはじめとしたサーバー側処理もコンポーネント責務にするなど、「必要なことは全てコンポーネントにカプセル化する」という方向性が強まっているように感じます。そしてレンダリングに境界を設け並行性を高めるのが`<Suspense>`です。これらを鑑みても、PPR は非常に昨今の React らしい設計と言えるのではないでしょうか。
