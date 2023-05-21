---
title: "Next.js App Router 知られざるClient-side cachingの仕様と実装"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

前回、App Routerの遷移の仕組みと実装についてまとめました。

https://zenn.dev/akfm/articles/next-app-router-navigation

今回はこれの続編として、App RouterのClient-side cachingについてまとめようと思います。

:::message
- 前回記事同様、細かい仕様や内部実装の話がほとんどで、機能の説明などは省略しているのでそちらは[公式ドキュメント](https://nextjs.org/docs)や他の記事をご参照ください。
- prefetch周りの仕様や実装は膨大なので、筆者の興味の向くままに大枠を調査したものです。
- 実装は当然ながらアップデートされるため、記事の内容が最新にそぐわない可能性があります。執筆時に見てたコミットは[afddb6e](https://github.com/vercel/next.js/tree/afddb6ebdade616cdd7780273be4cd28d4509890)です。
:::

## App Routerのキャッシュ分類

Next.jsのApp Routerは積極的にキャッシュを取り入れており、キャッシュは用途や段階に応じていくつかに分類することができます。

### Request Deduping

[Request Deduping](https://nextjs.org/docs/app/building-your-application/data-fetching#automatic-fetch-request-deduping)はレンダリングツリー内で同一データのGETリクエストを行う際に、自動でまとめてくれる機能です。

![](https://nextjs.org/_next/image?url=%2Fdocs%2Fdark%2Fdeduplicated-fetch-requests.png&w=3840&q=75)

デフォルトでサポートしているのはfetchのみですが、Reactが提供する`cache`を利用することでDBアクセスやGraphQLでも同様のキャッシュを実現することができます。

https://nextjs.org/docs/app/building-your-application/data-fetching/caching#react-cache

[この辺](https://nextjs.org/docs/app/building-your-application/data-fetching#the-fetch-api)を読むに、これはNext.jsではなくReact側で**fetchを拡張**することで行なっているようです。

以下の記事がRequest Dedupingについてより詳解されているので、興味のある方はぜひご一読ください。

https://zenn.dev/cybozu_frontend/articles/next-caching-dedupe

### Data fetching cache

Request Dedupingはレンダリングツリーを跨ぐキャッシュなのに対し、[Data fetching cache](https://nextjs.org/docs/app/building-your-application/data-fetching#caching-data)はCDN上などのlocation単位でキャッシュを保存します。

![](https://nextjs.org/_next/image?url=%2Fdocs%2Fdark%2Fstatic-site-generation.png&w=3840&q=75)

これらはどちらも`fetch`に対するキャッシュですが、前者はリクエスト処理単位のキャッシュであるのに対し後者はlocation単位のキャッシュです。

Data fetching cacheは[`revalidate`](https://nextjs.org/docs/app/building-your-application/data-fetching/revalidating)オプションや[`fetchCache`](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#fetchcache)を指定することで有効になります。

Data fetching cacheについても先述の記事を書かれた[mugiさん](https://zenn.dev/mugi)がシリーズ的に詳解されているので、こちらも興味のある方はぜひご一読ください。

https://zenn.dev/cybozu_frontend/articles/next-caching-revalidate

### CDN cache

App Routerでは`fetch`のキャッシュの話に目が行きがちですが、従来通り静的ファイルも[CDN cache](https://nextjs.org/docs/app/building-your-application/optimizing#static-assets)として扱えるよう設計されています。静的ファイル置き場である`public`フォルダはCDN cache可能なファイルとなることが想定され、VercelにデプロイするとデフォルトでCDN cacheされます。

### Client-side caching

[Client-side caching](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#client-side-caching-of-rendered-server-components)は、文字通りクライアントサイドのインメモリなキャッシュです。インメモリなのでリロードやMPA遷移を挟むと消えてしまいますが、App Router間の遷移においては有効です。

本稿の主題はこのキャッシュについてです。このキャッシュについてはNext.jsのドキュメントではあまり詳細に語られておらず、実装を見ないとわからない箇所もあるので本稿を執筆するに至った次第です。

## Client-side cachingの仕様と実装

Client-side cachingは内部的には`prefetchCache`と呼ばれているものです。文字通り**主に**prefetch時に格納されます。

[前回の記事](https://zenn.dev/akfm/articles/next-app-router-navigation)でも説明したように、App Routerは内部的に`useReducer`ベースで作成したStateで多くの状態を管理しており、`prefetchCache`もそのStateの一部として管理されています。具体的には`prefetchCache: Map<string, PrefetchCacheEntry>`の`data`にprefetchのPromiseごと格納しています。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/router-reducer-types.ts#L209-L215

このキャッシュは遷移発生時に利用され、そのままレンダリングされます。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/reducers/navigate-reducer.ts#L229

ちなみにキャッシュの取り出しは、Promiseを拡張して行なっているようです。少々行儀が悪い気がしますが、、、

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/create-record-from-thenable.ts

### Client-side cacheの種類

Client-side cacheには内部的に`auto`/`full`/`temporary`の3種類が存在します。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/router-reducer-types.ts#L165-L169

遷移時、`prefetchCache`に該当データがない時には`temporary`なprefetchが作成されます。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/reducers/navigate-reducer.ts#L204-L217

つまり、prefetchを無効にしてても内部的には`prefetchCache`は必要になるので、利用されるような作りになっています。

### Client-side cacheの有効期限

筆者が確認した限り、[ドキュメント](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#prefetching)には**有効期限の話やキャッシュをrevalidateする方法についての記載は見つけられませんでした**。ということで、実装から仕様を読み解いてみます。

Client-side cacheの生存期間は利用有無や`prefetch`のcacheの種類によって異なります。生存期間ごとのステータスは以下のenumで定義されています。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/get-prefetch-cache-entry-status.ts#L6-L11

このステータスは直後に定義されている関数によって判定が可能です。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/get-prefetch-cache-entry-status.ts#L18-L39

上記を読み解くと、Cacheの有効期限はざっくり以下のように分類されることがわかります。

- fresh: 新しいキャッシュ
  - prefetchから30s以内
- reusable: 再利用可能なキャッシュ
  - lastUsedから30s以内
- stale: ちょっと古いキャッシュ
  - prefetchから5m以内
- expired: 破棄されるべきキャッシュ
  - prefetchから5m以上

todo: 各cacheのDemo

#### 余談: stale時の挙動

flightにレンダリングされたコンポーネントを含まず、staleなときはprefetchを再度行うようなコメントや処理が見受けられましたが、どういう時にflightにレンダリングされたコンポーネントを含まないのか追いきれませんでした。

[このへん](https://github.com/vercel/next.js/pull/44502)が起点ぽいのですが、いまいち内容を把握しきれませんでした。後日気が向いたらまた調査してみようかと思います。

## Client-side cachingの問題点

TBW

## 構成

- とはいえこれでは`no-store`のように常に更新を促したい時にうまくいかない
- ではどうするべきか
  - 現状特にやりようはない
    - TODO ↓でやれる？
      - https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#invalidating-the-cache
  - 一方でこれでは普通に困ることもあるので、issueを立ててみました
    - TODO issue探してなければ作成
  - 以下はそこで提案してる内容です
    - キャッシュ時間を短くすると負荷問題が顕在化する
      - [ ] どのくらいprefetchするか試してみる
    - 暫定的にはprefetchをやめる
    - より根本的には、App Routerがクライアントサイドのキャッシュをコントロールできるように提供すべき？

