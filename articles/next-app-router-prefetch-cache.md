---
title: "Next.js App Router 知られざるClient-side cacheの仕様"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

前回、App Routerの遷移の仕組みと実装についてまとめました。

https://zenn.dev/akfm/articles/next-app-router-navigation

今回はこれの続編として、App RouterのClient-side cacheの仕様や実装についてまとめようと思います。まだドキュメントに記載のない仕様についても言及しているので、参考になる部分があれば幸いです。

:::message
- 前回記事同様、細かい仕様や内部実装の話がほとんどで、機能の説明などは省略しているのでそちらは[公式ドキュメント](https://nextjs.org/docs)や他の記事をご参照ください。
- 極力丁寧に説明するよう努めますが、前回の続きな部分も多いので[前回記事](https://zenn.dev/akfm/articles/next-app-router-navigation)から読むことをお勧めします。
- prefetch周りの仕様や実装は膨大なので、筆者の興味の向くままに大枠を調査したものです。
- 実装は当然ながらアップデートされるため、記事の内容が最新にそぐわない可能性があります。執筆時に見てたコミットは[afddb6e](https://github.com/vercel/next.js/tree/afddb6ebdade616cdd7780273be4cd28d4509890)です。
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

以下の記事がRequest Dedupingについてより詳解されているので、興味のある方はぜひご一読ください。

https://zenn.dev/cybozu_frontend/articles/next-caching-dedupe

### Caching Data

Request Dedupingは同一レンダリングツリー内で有効になるcacheなのに対し、[Caching Data](https://nextjs.org/docs/app/building-your-application/data-fetching#caching-data)はCDN上などのlocation単位でcacheを保存します。

![](https://nextjs.org/_next/image?url=%2Fdocs%2Fdark%2Fstatic-site-generation.png&w=3840&q=75)
*nextjs.org/docsより*

こちらも[同様の箇所](https://nextjs.org/docs/app/building-your-application/data-fetching#the-fetch-api)に記述があり、**Next.js側でfetchを拡張**しているようです。

Caching Dataは[`revalidate`](https://nextjs.org/docs/app/building-your-application/data-fetching/revalidating)オプションや[`fetchCache`](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#fetchcache)を指定することで有効になります。
こちらも先述の記事を書かれた[mugiさん](https://zenn.dev/mugi)がシリーズ的に解説されているので、こちらも興味のある方はぜひご一読ください。

https://zenn.dev/cybozu_frontend/articles/next-caching-revalidate

### CDN cache

App Routerでは`fetch`のcacheの話に目が行きがちですが、従来通り静的ファイルも[CDN cache](https://nextjs.org/docs/app/building-your-application/optimizing#static-assets)として扱えるよう設計されています。

静的ファイル置き場である`public`フォルダはCDN cache可能なファイルとなることが想定され、VercelにデプロイするとデフォルトでCDN cacheされます。また、[Static Site Generation (SSG)](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation#when-should-i-use-static-generation)した結果も同様に静的ファイルになるので、これらもCDNによってcacheすることが可能です。

### Client-side caching

[Client-side caching](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#client-side-caching-of-rendered-server-components)は、文字通りクライアントサイドのインメモリなcacheです。インメモリなのでリロードやMPA遷移を挟むと消えてしまいますが、App Router間の遷移においては有効です。

このcacheについてはNext.jsのドキュメントではあまり詳細に語られておらず、実装を見ないとわからない箇所もあるので、本稿はこのClient-side cacheについて実装を追いながら仕様を確認していきたいと思います。

## Client-side cacheの保存と利用

Client-side cacheはApp Router的にどう実装されているのでしょう？Client-side cacheは内部的には`prefetchCache`と呼ばれています。文字通りprefetch時に格納されるのですが、実は**prefetch以外**でも格納されます。詳細は後述しますので、まずは`prefetchCache`の実装を確認してみます。

[前回の記事](https://zenn.dev/akfm/articles/next-app-router-navigation)でも説明したように、App Routerは内部的に`useReducer`ベースで作成したStateで多くの状態を管理しており、`prefetchCache`もそのStateの一部として管理されています。具体的には`prefetchCache: Map<string, PrefetchCacheEntry>`の`data`にprefetchのPromiseごと格納しています。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/router-reducer-types.ts#L209-L215

このcacheは遷移時に発火する`navigate`アクションのreducer内で読み取られます。`prefetchCache`という命名が紛らわしいのですが、cacheがなかった場合、fetchを行ってこのcacheを作成するなど、**App Routerの遷移には必ず`prefetchCache`が必要**になります。（なぜ`prefetchCache`という命名なのかは不明ですが、開発中に仕様が変わり続けて残ってしまったのかもしれません）

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/reducers/navigate-reducer.ts#L229

ちなみにPromiseから値を同期的に読み取る`readRecordValue`は、Promiseを拡張して行なっているようです。（これも少々行儀が悪い気がしますが、、、）

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/create-record-from-thenable.ts

## Client-side cacheの種別

さて、Client-side cacheには内部的に`auto`/`full`/`temporary`の3種類が存在します。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/router-reducer-types.ts#L158-L169

コメントに説明があるのでDeepl翻訳してみます。

- `auto` - ページが動的な場合は、ページデータを部分的にプリフェッチし、静的な場合はページデータを完全にプリフェッチします。
- `full` - ページデータを完全にプリフェッチする。 
- `temporary` - これは next/link で prefetch={false} が使われているときや、プログラムでルートをプッシュするときに使用されます。

ここで言う動的なページとは、リクエストが来るまでレンダリングできないページを指します。App Routerは動的なページかどうか、[dynamic functions](https://nextjs.org/docs/app/building-your-application/rendering/static-and-dynamic-rendering#using-dynamic-functions)（[cookies](https://nextjs.org/docs/app/api-reference/functions/cookies)や[headers](https://nextjs.org/docs/app/api-reference/functions/headers)など）が使われているかどうかで判断されます。`next build`時に一度各pageコンポーネントを実行してるので、その時に判断されるものと考えられます。

また、`full`と`temporary`は明示的に`prefetch`を`Link`コンポーネントに渡すことで指定できます。`auto`は`prefetch`を指定しない場合のデフォルト値、`full`は`prefetch={true}`、`temporary`は`prefetch={false}`です。

以下の簡単なデモページで挙動を確認してみましょう。`Links`にあるリンクはそれぞれ、キャッシュの種類ごとにページリンクになります。`auto`のみ動的ページかどうかで分岐があるので、2ページ用意しています。

- `/cache_auto/static`: キャッシュ種別が`auto`な動的**でない**ページへのリンク
- `/cache_auto/dynamic`: キャッシュ種別が`auto`な動的ページ（`next/headers`を利用）へのリンク
- `/cache_full`: キャッシュ種別が`full`（`prefetch={true}`）なリンク
- `/cache_temporary`: キャッシュ種別が`temporary`（`prefetch={false}`）なリンク

![](/images/next-app-router-prefetch-cache/demo-prefetch-list.png)
*画面内に要素があるとprefetchが発火する*

viewport内に`Link`が入ってくると`prefetch`アクションが発火し、リンク先ページのレンダリング結果を取得しようと試みます。`prefetch={false}`な`/cache_temporary`は当然prefetchされないので他の3ページのprefetchが確認できます。

![](/images/next-app-router-prefetch-cache/demo-prefetch-static.png)
*ページが動的でない時はレンダリング結果がflightで送られてくる*

`/cache_auto/static`のレスポンスBodyを確認すると**Flight**が確認できます。Flightについては[前回の記事](https://zenn.dev/akfm/articles/next-app-router-navigation#react-flight)でも言及していますが、React Server Components（RSC）をレンダリングした結果を表現する、独自のデータフォーマットです。

![](/images/next-app-router-prefetch-cache/demo-prefetch-dynamic.png)
*ページ全体が動的な時はレンダリングできないので空になる*

一方で`/cache_auto/dynamic`の方は中身が空になってることが確認できます。このページの実装は以下のようになっています。

```tsx
// /src/app/cache_auto/dynamic/page.tsx
export default async function Page() {
  const res = await fetch("https://dummyjson.com/products");
  await timer();
  const data = await res.json();
  // dynamic functions
  const headersList = headers();
  console.log("headersList", headersList);

  return (
    ...
  );
}
```

`page`自体がdynamic functionsに依存しています。`layout.tsx`もないので、レンダリングできるものがなく空が帰ったものと考えられます。

![](/images/next-app-router-prefetch-cache/demo-prefetch-full.png)

`/cache_full`は`/cache_auto/static`と特段変わった様子はないようです。`full`の大きな違いはClient-side cacheの有効時間にあるためです。

### Client-side cacheの有効期限

cacheというからには当然ながら有効期限があり、前述のcacheの種別によって有効期限の仕様が異なります。内部的にはこれはステータスとして管理されており、以下のenumで定義されています。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/get-prefetch-cache-entry-status.ts#L6-L11

上記よりcacheのステータスは、以下のように分類されることがわかります。

- `fresh`: 新しいcache
- `reusable`: 再利用可能なcache
- `stale`: ちょっと古いcache
- `expired`: 破棄されるべきcache

このステータスは直後に定義されている関数によって内部的に判定されます。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/get-prefetch-cache-entry-status.ts#L18-L39

上記関数より、cacheは種別・経過時間・`lastUsed`（cacheの最後の利用時間）によって以下のようなステータス分類が行われていることがわかります。

| cache種別                | `auto` | `full` | `temporary` |
| ----------------------- | ------- | ---- | ---- |
| prefetchから**30秒以内**                  | `fresh`    | `fresh` | `fresh` |
| lastUsedから**30秒以内**     | `reusable` | `reusable` | `reusable` |
| prefetchから**30秒~5分**                  | `stale`    | `reusable` | `expired` |
| prefetchから**5分~30分**        | `expired`    | `reusable` | `expired` |
| prefetchから**30分以降**                  | `expired`    | `expired` | `expired` |

これらについてもそれぞれ挙動や実装を確認してみると、prefetchに以下のような違いがあるようでした。

- `fresh`, `reusable`: prefetchを再発行せず、cacheを再利用する
- `stale`: dynamic functionsを含むレンダリング部分だけprefetchを再度行う
- `expired`: prefetchを再発行する

:::message
厳密には各ステータスと状況ごとに多くの分岐が存在するので、上記はあくまで基本的な挙動とご理解ください。
:::

### Client-side cacheのrevalidate

しかし、cacheというからには当然任意のタイミングでrevalidateしたいケースが存在するであろうことが想像できます。

筆者が確認した限り、[ドキュメント](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#prefetching)には**有効期限の話やcacheをrevalidateする方法についての記載は見つけられませんでした**。`fetch`に`cache: 'no-store'`がついてたり、`prefetch={false}`だったとしても、`fresh`や`reusable`なcacheは再利用されてしまいます。

[On-Demand Revalidation](https://nextjs.org/docs/app/building-your-application/data-fetching/revalidating#using-on-demand-revalidation)の機能なども試してみましたが、これがクリアできるcacheはやはり`Caching Data`などが対象のようで、Client-side cacheをクリアすることはできませんでした。

これについてはすでにいくつかissueが立っており、以下のissueが最も盛んに議論されているようでした。

https://github.com/vercel/next.js/issues/42991

上記issueでは`router.refresh()`を利用する手段などが提案されており、試したところ確かに毎回cacheを利用せず再度fetchしている様子が確認できました。ただ、これはスマートな解決策とは言い難いところです。できることならClient-side cacheの時間を任意に指定できたり、動的ページの場合はデフォルトで`router.refresh()`を呼び出してくれたり、任意のタイミングでrevalidateする手段が提供されていることが望ましい気がします。

今後何かしらの対応がされ、ドキュメントも更新されることを願います。

## 感想

今回はClient-side cacheに重きを置いて実装や仕様を調査してみました。App Routerの積極的なキャッシュは[INP](https://web.dev/inp/)（**Interaction To Next Paint**）などの改善が見込まれるし、歓迎してる部分も多いのです。一方でClient-side cacheが一定時間保持され続けてしまうことについては、ドキュメントでの説明や対応方法の不足があり、プロダクションで利用するにはかなり大きな制約になると感じています。特にユーザー情報を扱うWebアプリ開発などでは、この手の問題は致命的になり得ます。

App Routerは注目度も高く、すでにstableが宣言されたわけですが、利用する開発者側からすると挙動や機能もまだ「安定」していない部分があると筆者は考えています。App Routerはユーザーにとっても開発者にとっても多くのメリットがあるし、非常に魅力的な機能を兼ねているので、上記問題含めさらに多くの改善や発展に期待したいところです。
