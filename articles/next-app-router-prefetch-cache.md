---
title: "Next.js App Router 知られざるClient-side Cacheの仕様"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

前回、App Routerの遷移の仕組みと実装についてまとめました。

https://zenn.dev/akfm/articles/next-app-router-navigation

今回はこれの続編として、App RouterのClient-side Cacheの仕様や実装についてまとめようと思います。まだドキュメントに記載のない仕様についても言及しているので、参考になる部分があれば幸いです。

:::message
- 前回記事同様、細かい仕様や内部実装の話がほとんどで、機能の説明などは省略しているのでそちらは[公式ドキュメント](https://nextjs.org/docs)や他の記事をご参照ください。
- 極力丁寧に説明するよう努めますが、前回の続きな部分も多いのでより深く理解したい方は[こちらの記事](https://zenn.dev/akfm/articles/next-app-router-navigation)から読むことをお勧めします。
- prefetch周りの仕様や実装は膨大なので、筆者の興味の向くままに大枠を調査したものです。
- 実装は当然ながらアップデートされるため、記事の内容が最新にそぐわない可能性があります。調査・執筆時に見てたコミットは[afddb6e](https://github.com/vercel/next.js/tree/afddb6ebdade616cdd7780273be4cd28d4509890)です。
:::

## App Routerのcache分類

App Routerは積極的にcacheを取り入れており、cacheは用途や段階に応じていくつかに分類することができます。まずはそのcacheの分類を確認してみましょう。

### Request Deduping

[Request Deduping](https://nextjs.org/docs/app/building-your-application/data-fetching#automatic-fetch-request-deduping)はレンダリングツリー内で同一データのGETリクエストを行う際に、自動でまとめてくれる機能です。

![](https://nextjs.org/_next/image?url=%2Fdocs%2Fdark%2Fdeduplicated-fetch-requests.png&w=3840&q=75)
*nextjs.org/docsより*

デフォルトでサポートしているのはfetchのみですが、Reactが提供する`cache`を利用することでDBアクセスやGraphQLでも同様のcacheを実現することができます。

https://nextjs.org/docs/app/building-your-application/data-fetching/caching#react-cache

[この辺](https://nextjs.org/docs/app/building-your-application/data-fetching#the-fetch-api)を読むに、これはNext.jsではなく**React側でfetchを拡張**することで行なっているようです。

以下の記事がRequest Dedupingについてより詳しく解説されているので、興味のある方はぜひご一読ください。

https://zenn.dev/cybozu_frontend/articles/next-caching-dedupe

### Caching Data

[Caching Data](https://nextjs.org/docs/app/building-your-application/data-fetching#caching-data)はRequest Deduping同様、主にfetchによるGETリクエストが対象ですが、Request Dedupingは同一レンダリングツリー内で有効になるcacheなのに対し、Caching DataはCDN上などのlocation単位でcacheを保存します。Request Dedupingは1リクエストに対し、Caching Dataは複数リクエストに対し有効と考えるとわかりやすいかと思います。

![](https://nextjs.org/_next/image?url=%2Fdocs%2Fdark%2Fstatic-site-generation.png&w=3840&q=75)
*nextjs.org/docsより*

これはReact側ではなく、[Next.js側でfetchを拡張](https://nextjs.org/docs/app/building-your-application/data-fetching#the-fetch-api)することで実現しているようです。

Caching Dataは[`revalidate`](https://nextjs.org/docs/app/building-your-application/data-fetching/revalidating)オプションや[`fetchCache`](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#fetchcache)を指定することで有効になります。
こちらも先述の記事を書かれた[mugiさん](https://zenn.dev/mugi)がシリーズ的に解説されているので、興味のある方はぜひご一読ください。

https://zenn.dev/cybozu_frontend/articles/next-caching-revalidate

### CDN cache

App Routerでは`fetch`のcacheの話に目が行きがちですが、従来通り静的ファイルも[CDN cache](https://nextjs.org/docs/app/building-your-application/optimizing#static-assets)として扱えるよう設計されています。

静的ファイル置き場である`public`フォルダはCDN cache可能なファイルとなることが想定され、VercelにデプロイするとデフォルトでCDN cacheされます。また、[Static Site Generation (SSG)](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation#when-should-i-use-static-generation)した結果も同様に静的ファイルになるので、これらもCDNにcacheすることが可能です。

### Client-side caching

上記3つのcacheはユーザーからするとサーバー（主にCDNですが）側のcacheです。対して、[Client-side caching](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#client-side-caching-of-rendered-server-components)は、文字通り**クライアントサイドのインメモリなcache**です。インメモリなのでリロードやMPA遷移を挟むと消えてしまいますが、App Router間の遷移においては有効です。

このcacheについてはNext.jsのドキュメントではあまり詳細に語られておらず、筆者が確認した限りだと以下くらいの情報しかありません。

> The new router has an in-memory client-side Cache that stores the rendered result of Server Components (payload). The cache is split by route segments which allows invalidation at any level and ensures consistency across concurrent renders.
> 
> As users navigate around the app, the router will store the payload of previously fetched segments and prefetched segments in the cache.
> 
> This means, for certain cases, the router can re-use the cache instead of making a new request to the server. This improves performance by avoiding re-fetching data and re-rendering components unnecessarily.
> <以下Deepl訳>
> 新しいルーターには、Server Componentsのレンダリング結果（ペイロード）を保存するインメモリ・クライアントサイド・キャッシュがあります。キャッシュはルートセグメントで分割されているため、どのレベルでも無効化でき、同時レンダリング時の一貫性を確保できます。
> 
> ユーザーがアプリ内を移動すると、ルーターは以前にフェッチしたセグメントとプリフェッチしたセグメントのペイロードをキャッシュに保存します。
> 
> これは、特定のケースにおいて、ルーターがサーバーに新たなリクエストを行う代わりにキャッシュを再利用できることを意味します。これにより、不必要なデータの再取得やコンポーネントの再レンダリングを回避し、パフォーマンスを向上させることができます。

上記内容は「キャッシュの無効化」の説明や「特定のケース」の定義など、多くの点で説明が省略されてしまってて詳細がわかりません。以降はこのClient-side Cacheについて、実装を追いながら説明が省略されている仕様を確認していきたいと思います。

## Client-side Cacheの保存と利用

Client-side CacheはApp Router的にどう実装されているのでしょう？Client-side Cacheは内部的には`prefetchCache`と呼ばれています。文字通りprefetch時に格納されるのですが、実は**prefetch以外**でも格納されます。なぜ`prefetchCache`という命名なのかは不明ですが、開発中に仕様が変わり続けて残ってしまったのかもしれません。詳細は後述しますので、まずは`prefetchCache`の実装を確認してみます。

[前回の記事](https://zenn.dev/akfm/articles/next-app-router-navigation)でも説明したように、App Routerは内部的に`useReducer`ベースで作成したStateで多くの状態を管理しており、`prefetchCache`もそのStateの一部として管理されています。具体的には`prefetchCache: Map<string, PrefetchCacheEntry>`の`data`にprefetchの**Promiseごと格納**しています。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/router-reducer-types.ts#L209-L215

このcacheは遷移時に発火する`navigate`アクションのreducer内で読み取られます。`prefetchCache`という命名が紛らわしいのですが、cacheがなかった場合、fetchを行ってこのcacheを作成するなど、**App Routerの遷移には必ず`prefetchCache`が必要**になります。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/reducers/navigate-reducer.ts#L229

ちなみにPromiseから値を同期的に読み取る`readRecordValue`は、Promiseを拡張して行なっているようです。（これも少々行儀が悪い気がしますが、、、）

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/create-record-from-thenable.ts

## Client-side Cacheの種別

さて、Client-side Cacheには内部的に`auto`/`full`/`temporary`の3種類が存在します。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/router-reducer-types.ts#L158-L169

コメントに説明があるのでDeepl翻訳してみます。

- `auto` - ページが動的な場合は、ページデータを部分的にプリフェッチし、静的な場合はページデータを完全にプリフェッチします。
- `full` - ページデータを完全にプリフェッチする。 
- `temporary` - これは`next/link`で`prefetch={false}`が使われているときや、プログラムでルートをプッシュするときに使用されます。

ここで言う動的なページとは[Dynamic Rendering](https://nextjs.org/docs/app/building-your-application/rendering/static-and-dynamic-rendering#dynamic-rendering)と呼ばれる、リクエストが来るまでレンダリングできないページを指します。App RouterはDynamic Renderingなページかどうかを`no-store`なfetchがあるかどうかや、[Dynamic Functions](https://nextjs.org/docs/app/building-your-application/rendering/static-and-dynamic-rendering#using-dynamic-functions)（[cookies](https://nextjs.org/docs/app/api-reference/functions/cookies)・[headers](https://nextjs.org/docs/app/api-reference/functions/headers)など）が使われているかどうかで判断されます。`next build`時に一度各pageコンポーネントを実行してるので、その時に判断されるものと考えられます。

これらのcache種別は基本的に`Link`コンポーネントの`prefetch`と対応関係にあります。

- `auto`: `prefetch={undefined}`
- `full`: `prefetch={true}`
- `temporary`: `prefetch={false}`

以下の簡単なデモページで挙動を確認してみましょう。`Links`にあるリンクはそれぞれ、cache種別ごとにページリンクになります。`auto`のみDynamic Renderingかどうかで分岐があるので、2ページ用意しています。

| URL | prefetch | Dynamic Rendering | cache種別 |
| ---- | ---- | ---- | ---- |
| `/cache_auto/static` | `undefined` | false | `auto` |
| `/cache_auto/dynamic` | `undefined` | true（`next/headers`を利用） | `auto` |
| `/cache_full` | `true` | false | `full` |
| `/cache_temporary` | `false` | false | `temporary` |

![](/images/next-app-router-prefetch-cache/demo-prefetch-list.png)
*画面内に要素があるとprefetchが発火する*

viewport内に`Link`が入ってくると`prefetch`アクションが発火し、リンク先ページのレンダリング結果を取得しようと試みます。`prefetch={false}`な`/cache_temporary`は当然prefetchされないので他の3ページのprefetchが確認できます。

![](/images/next-app-router-prefetch-cache/demo-prefetch-static.png)
*ページが動的でない時はレンダリング結果がflightで送られてくる*

`/cache_auto/static`のレスポンスBodyを確認すると**Flight**が確認できます。Flightについては[前回の記事](https://zenn.dev/akfm/articles/next-app-router-navigation#react-flight)でも言及していますが、React Server Components（RSC）をレンダリングした結果を表現する、独自のデータフォーマットです。

![](/images/next-app-router-prefetch-cache/demo-prefetch-dynamic.png)
*ページ全体が動的な時はレンダリングできないので他ページよりレンダリング結果が減っている*

`/cache_auto/dynamic`ページの実装は以下のようになっています。

```tsx
// /src/app/cache_auto/dynamic/page.tsx
export default async function Page() {
  const res = await fetch("https://dummyjson.com/products");
  await timer();
  const data = await res.json();
  // Dynamic Functions
  const headersList = headers();
  console.log("headersList", headersList);

  return (
    ...
  );
}
```

`page`がDynamic Functions（`next/headers`）に依存しているため、このページはDynamic Renderingと判定されます。

以下は実際のレスポンスBodyです。ページの内容的には`/cache_auto/static`より多いはずですが、Dynamic Renderingはprefetch時点ではレンダリングされないため、`/cache_auto/dynamic`の方がレンダリング結果が少ないことが確認できます。

```
1:HL["/_next/static/css/2a761abac1cc65a8.css",{"as":"style"}]
0:[[["",{"children":["cache_auto",{"children":["static",{"children":["__PAGE__",{}]}]}]},"$undefined","$undefined",true],"$L2",[[["$","link","0",{"rel":"stylesheet","href":"/_next/static/css/2a761abac1cc65a8.css","precedence":"next"}]],["$L3",null]]]]
4:I{"id":"663","chunks":["272:static/chunks/webpack-6cd0790319613630.js","602:static/chunks/c3978839-04849c4dd0c49610.js","269:static/chunks/269-211c86a5f099d56d.js"],"name":"","async":false}
5:I{"id":"6712","chunks":["272:static/chunks/webpack-6cd0790319613630.js","602:static/chunks/c3978839-04849c4dd0c49610.js","269:static/chunks/269-211c86a5f099d56d.js"],"name":"","async":false}
6:I{"id":"863","chunks":["863:static/chunks/863-a23d5de87448ddb7.js","919:static/chunks/app/cache_auto/dynamic/page-d873c36f84943bee.js"],"name":"","async":false}
2:[["$","html",null,{"lang":"en","children":["$","body",null,{"children":["$","div",null,{"className":"flex min-h-screen flex-col items-center justify-between p-24","children":["$","div",null,{"className":"w-full max-w-5xl","children":["$","$L4",null,{"parallelRouterKey":"children","segmentPath":["children"],"error":"$undefined","errorStyles":"$undefined","loading":"$undefined","loadingStyles":"$undefined","hasLoading":false,"template":["$","$L5",null,{}],"templateStyles":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","asNotFound":"$undefined","childProp":{"current":["$","$L4",null,{"parallelRouterKey":"children","segmentPath":["children","cache_auto","children"],"error":"$undefined","errorStyles":"$undefined","loading":"$undefined","loadingStyles":"$undefined","hasLoading":false,"template":["$","$L5",null,{}],"templateStyles":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","asNotFound":"$undefined","childProp":{"current":["$","$L4",null,{"parallelRouterKey":"children","segmentPath":["children","cache_auto","children","static","children"],"error":"$undefined","errorStyles":"$undefined","loading":"$undefined","loadingStyles":"$undefined","hasLoading":false,"template":["$","$L5",null,{}],"templateStyles":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","asNotFound":"$undefined","childProp":{"current":[["$","main",null,{"children":[["$","h1",null,{"className":"mb-4 text-3xl font-extrabold text-gray-900 dark:text-white md:text-5xl lg:text-6xl","children":[[["$","span",null,{"className":"text-transparent bg-clip-text bg-gradient-to-r to-emerald-600 from-sky-400","children":"`app`"}],"Â "],"cache auto static"]}],["$","p",null,{"className":"text-lg font-normal text-gray-500 lg:text-xl dark:text-gray-400","children":"This is an example page."}],["$","div",null,{"className":"mt-10","children":[["$","h2",null,{"className":"mb-4 text-xl font-extrabold text-gray-900 dark:text-white md:text-4xl lg:text-4xl","children":"Links"}],["$","ul",null,{"className":"list-disc pl-5","children":[["$","li",null,{"children":["$","$L6",null,{"href":"/cache_auto/static","className":"underline","children":"/cache_auto/static"}]}],["$","li",null,{"children":["$","$L6",null,{"href":"/cache_auto/dynamic","className":"underline","children":"/cache_auto/dynamic"}]}],["$","li",null,{"children":["$","$L6",null,{"href":"/cache_full","className":"underline","prefetch":true,"children":"/cache_full"}]}],["$","li",null,{"children":["$","$L6",null,{"href":"/cache_temporary","className":"underline","prefetch":false,"children":"/cache_temporary"}]}]]}]]}]]}],null],"segment":"__PAGE__"},"styles":[]}],"segment":"static"},"styles":[]}],"segment":"cache_auto"},"styles":[]}]}]}]}]}],null]
3:[[["$","meta",null,{"charSet":"utf-8"}],["$","title",null,{"children":"Create Next App"}],["$","meta",null,{"name":"description","content":"Generated by create next app"}],null,null,null,null,null,null,null,null,["$","meta",null,{"name":"viewport","content":"width=device-width, initial-scale=1"}],null,null,null,null,null,null,null,null,null,null,[]],[null,null,null,null],null,null,[null,null,null,null,null],null,null,null,null,[null,[["$","link",null,{"rel":"icon","href":"/favicon.ico","type":"image/x-icon","sizes":"any"}]],[],null]]
```
*`/cache_auto/static`のFlight*

```
0:[["children","cache_auto",["cache_auto",{"children":["dynamic",{"children":["__PAGE__",{}]}]}],"$L1",[[],["$L2",null]]]]
3:I{"id":"663","chunks":["272:static/chunks/webpack-6cd0790319613630.js","602:static/chunks/c3978839-04849c4dd0c49610.js","269:static/chunks/269-211c86a5f099d56d.js"],"name":"","async":false}
4:I{"id":"6712","chunks":["272:static/chunks/webpack-6cd0790319613630.js","602:static/chunks/c3978839-04849c4dd0c49610.js","269:static/chunks/269-211c86a5f099d56d.js"],"name":"","async":false}
1:["$","$L3",null,{"parallelRouterKey":"children","segmentPath":["children","cache_auto","children"],"error":"$undefined","errorStyles":"$undefined","loading":"$undefined","loadingStyles":"$undefined","hasLoading":false,"template":["$","$L4",null,{}],"templateStyles":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","asNotFound":"$undefined","childProp":{"current":["$","$L3",null,{"parallelRouterKey":"children","segmentPath":["children","cache_auto","children","dynamic","children"],"loading":"loading...","loadingStyles":[],"hasLoading":true,"error":"$undefined","errorStyles":"$undefined","template":["$","$L4",null,{}],"templateStyles":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","childProp":{"current":null,"segment":"__PAGE__"}}],"segment":"dynamic"},"styles":[]}]
2:[[["$","meta",null,{"charSet":"utf-8"}],["$","title",null,{"children":"Create Next App"}],["$","meta",null,{"name":"description","content":"Generated by create next app"}],null,null,null,null,null,null,null,null,["$","meta",null,{"name":"viewport","content":"width=device-width, initial-scale=1"}],null,null,null,null,null,null,null,null,null,null,[]],[null,null,null,null],null,null,[null,null,null,null,null],null,null,null,null,[null,[["$","link",null,{"rel":"icon","href":"/favicon.ico","type":"image/x-icon","sizes":"any"}]],[],null]]
```
*`/cache_auto/dynamic`のFlight*

![](/images/next-app-router-prefetch-cache/demo-prefetch-full.png)

`/cache_full`は`/cache_auto/static`と特段変わった様子はないようです。`full`の大きな違いはClient-side Cacheの有効時間にあるためです。

### Client-side Cacheの有効期限

cacheというからには当然ながら有効期限があり、前述のcacheの種別によって有効期限の仕様が異なります。内部的にはこれはステータスとして管理されており、以下のenumで定義されています。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/get-prefetch-cache-entry-status.ts#L6-L11

上記よりcacheのステータスは、以下のように分類されることがわかります。

- `fresh`: 新しいcache
- `reusable`: 再利用可能なcache
- `stale`: 古いcache
- `expired`: 破棄されるべきcache

このステータスは直後に定義されている関数によって内部的に判定されます。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/get-prefetch-cache-entry-status.ts#L18-L39

上記関数より、cacheは種別・経過時間・`lastUsed`（cacheの最後の利用時間）によって以下のようなステータス分類が行われていることがわかります。

| 時間判定                | `auto` | `full` | `temporary` |
| ----------------------- | ------- | ---- | ---- |
| prefetchから**30秒以内**                  | `fresh`    | `fresh` | `fresh` |
| lastUsedから**30秒以内**     | `reusable` | `reusable` | `reusable` |
| prefetchから**30秒~5分**                  | `stale`    | `reusable` | `expired` |
| prefetchから**5分~30分**        | `expired`    | `reusable` | `expired` |
| prefetchから**30分以降**                  | `expired`    | `expired` | `expired` |

これらについてもそれぞれ挙動や実装を確認してみると、prefetchに以下のような違いがあるようでした。

- `fresh`, `reusable`: prefetchを再発行せず、cacheを再利用する
- `stale`: Dynamic Rendering部分（segment）だけ遷移時に再fetchを行う
- `expired`: prefetchを再発行する

:::message
厳密には各ステータスと状況ごとに多くの分岐が存在するので、上記はあくまで基本的な挙動とご理解ください。
:::

### Client-side Cacheのrevalidate

しかし、cacheというからには当然任意のタイミングでrevalidateしたいケースが存在するであろうことが想像できます。

筆者が確認した限り、[ドキュメント](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#prefetching)には**有効期限の話やrevalidateする方法についての記載は見つけられませんでした**。Dynamic Renderingなページでも、`fresh`や`reusable`になりうるため、cacheがあれば再利用されてしまいます。

[On-Demand Revalidation](https://nextjs.org/docs/app/building-your-application/data-fetching/revalidating#using-on-demand-revalidation)の機能なども試してみましたが、これがクリアできるcacheはやはりサーバー側（Caching Data）が対象のようで、Client-side Cacheをクリアすることはできませんでした。

これについてはすでにいくつかissueが立っており、以下のissueが最も盛んに議論されているようでした。

https://github.com/vercel/next.js/issues/42991

上記issueでは`router.refresh()`を利用する手段などが提案されており、試したところ、確かに毎回cacheを利用せず再度fetchしている様子が確認できました。ただ、これはスマートな解決策とは言い難いところです。できることならClient-side Cacheの時間を任意に指定できたり、Dynamic Renderingの場合はデフォルトで`router.refresh()`を呼び出してくれたり、任意のタイミングでrevalidateする手段が提供されていることが望ましい気がします。また、おそらくですがページ全体をrefreshしてしまうので、ドキュメントにあった

> キャッシュはルートセグメントで分割されているため、どのレベルでも無効化でき

というのはNext.js内部的な話だけで、開発者がセグメントレベルで無効化できるわけではないものと考えられます。

せめて現状の仕様については記載が欲しいので、こちらはissueを立てました。

https://github.com/vercel/next.js/issues/49431

今後何かしらの対応がされ、ドキュメントも更新されることを願います。

## 感想

今回はClient-side Cacheに重きを置いて実装や仕様を調査してみました。App Routerの積極的なキャッシュは[INP](https://web.dev/inp/)（**Interaction To Next Paint**）などの改善が見込まれるし、歓迎してる部分も多いのです。一方でClient-side Cacheが一定時間保持され続けてしまうことについては、ドキュメントでの説明や対応方法の不足があり、プロダクションで利用するにはかなり大きな制約になると感じています。特にユーザー情報を扱うWebアプリ開発などでは、この手の問題は致命的になり得ます。

App Routerは注目度も高く、すでにstableが宣言されたわけですが、利用する開発者側からすると挙動や機能もまだ「安定」していない部分があると筆者は考えています。App Routerはユーザーにとっても開発者にとっても多くのメリットがあるし、非常に魅力的な機能を兼ねているので、上記問題含めさらに多くの改善や発展に期待したいところです。
