---
title: "Next.js App Router 知られざるClient-side cacheの仕様"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

前回、App Routerの遷移の仕組みと実装についてまとめました。

https://zenn.dev/akfm/articles/next-app-router-navigation

今回はこれの続編として、App RouterのClient-side cacheの仕様や実装についてまたまとめようと思います。まだドキュメントに記載のない仕様についても言及しているので、参考になる部分があれば幸いです。

:::message
- 前回記事同様、細かい仕様や内部実装の話がほとんどで、機能の説明などは省略しているのでそちらは[公式ドキュメント](https://nextjs.org/docs)や他の記事をご参照ください。
- prefetch周りの仕様や実装は膨大なので、筆者の興味の向くままに大枠を調査したものです。
- 実装は当然ながらアップデートされるため、記事の内容が最新にそぐわない可能性があります。執筆時に見てたコミットは[afddb6e](https://github.com/vercel/next.js/tree/afddb6ebdade616cdd7780273be4cd28d4509890)です。
:::

## App Routerのcache分類

App Routerは積極的にcacheを取り入れており、cacheは用途や段階に応じていくつかに分類することができます。まずはそのcacheの分類を確認してみましょう。

### Request Deduping

[Request Deduping](https://nextjs.org/docs/app/building-your-application/data-fetching#automatic-fetch-request-deduping)はレンダリングツリー内で同一データのGETリクエストを行う際に、自動でまとめてくれる機能です。

![](https://nextjs.org/_next/image?url=%2Fdocs%2Fdark%2Fdeduplicated-fetch-requests.png&w=3840&q=75)
*nextjs.orgより*

デフォルトでサポートしているのはfetchのみですが、Reactが提供する`cache`を利用することでDBアクセスやGraphQLでも同様のcacheを実現することができます。

https://nextjs.org/docs/app/building-your-application/data-fetching/caching#react-cache

[この辺](https://nextjs.org/docs/app/building-your-application/data-fetching#the-fetch-api)を読むに、これはNext.jsではなくReact側で**fetchを拡張**することで行なっているようです。

以下の記事がRequest Dedupingについてより詳解されているので、興味のある方はぜひご一読ください。

https://zenn.dev/cybozu_frontend/articles/next-caching-dedupe

### Data fetching cache

Request Dedupingはレンダリングツリーを跨ぐcacheなのに対し、[Data fetching cache](https://nextjs.org/docs/app/building-your-application/data-fetching#caching-data)はCDN上などのlocation単位でcacheを保存します。

![](https://nextjs.org/_next/image?url=%2Fdocs%2Fdark%2Fstatic-site-generation.png&w=3840&q=75)
*nextjs.orgより*

これらはどちらも`fetch`に対するcacheですが、前者はリクエスト処理単位のcacheであるのに対し後者はlocation単位のcacheです。

Data fetching cacheは[`revalidate`](https://nextjs.org/docs/app/building-your-application/data-fetching/revalidating)オプションや[`fetchCache`](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#fetchcache)を指定することで有効になります。

Data fetching cacheについても先述の記事を書かれた[mugiさん](https://zenn.dev/mugi)がシリーズ的に解説されているので、こちらも興味のある方はぜひご一読ください。

https://zenn.dev/cybozu_frontend/articles/next-caching-revalidate

### CDN cache

App Routerでは`fetch`のcacheの話に目が行きがちですが、従来通り静的ファイルも[CDN cache](https://nextjs.org/docs/app/building-your-application/optimizing#static-assets)として扱えるよう設計されています。

静的ファイル置き場である`public`フォルダはCDN cache可能なファイルとなることが想定され、VercelにデプロイするとデフォルトでCDN cacheされます。

また、[Static Site Generation (SSG)](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation#when-should-i-use-static-generation)した結果も同様に静的ファイルになるので、これらもCDNによってcacheすることが可能です。

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

## Client-side cacheの種類

さて、Client-side cacheには内部的に`auto`/`full`/`temporary`の3種類が存在します。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/router-reducer-types.ts#L158-L169

コメントに説明があるのでDeepl翻訳してみます。

- `auto` - ページが動的な場合は、ページデータを部分的にプリフェッチし、静的な場合はページデータを完全にプリフェッチします。
- `full` - ページデータを完全にプリフェッチする。 
- `temporary` - これは next/link で prefetch={false} が使われているときや、プログラムでルートをプッシュするときに使用されます。

TBW: それぞれのDemo作ってキャプチャ貼る

### Client-side cacheの有効期限

遷移にはClient-side cacheが必要なこと、Client-side cacheにはいくつかの種類があることがわかりました。しかし、cacheというからには当然有効期限や失効させたい要件などが存在することが想像できます。

筆者が確認した限り、[ドキュメント](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#prefetching)には**有効期限の話やcacheをrevalidateする方法についての記載は見つけられませんでした**。ということで、実装から仕様を読み解いてみます。

Client-side cacheの生存期間は利用有無や`prefetch`のcacheの種類によって異なります。生存期間ごとのステータスは以下のenumで定義されています。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/get-prefetch-cache-entry-status.ts#L6-L11

このステータスは直後に定義されている関数によって判定が可能です。

https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/get-prefetch-cache-entry-status.ts#L18-L39

上記を読み解くと、Cacheの有効期限はざっくり以下のように分類されることがわかります。

- fresh: 新しいcache
  - prefetchから30s以内
- reusable: 再利用可能なcache
  - lastUsedから30s以内
- stale: ちょっと古いcache
  - prefetchから5m以内
- expired: 破棄されるべきcache
  - prefetchから5m以上

これらについても、それぞれ挙動を確認してみましょう。

TBW: 各cacheのDemo

#### 余談: stale時の挙動

flightにレンダリングされたコンポーネントを含まず、staleなときはprefetchを再度行うようなコメントや処理が見受けられましたが、どういう時にflightにレンダリングされたコンポーネントを含まないのか追いきれませんでした。

[このへん](https://github.com/vercel/next.js/pull/44502)が起点ぽいのですが、いまいち内容を把握しきれませんでした。後日気が向いたらまた調査してみようかと思います。

## Client-side cacheの問題点

TBW

- とはいえこれでは`no-store`のように常に更新を促したい時にうまくいかない
- ではどうするべきか
  - 現状特にやりようはない
    - TODO ↓でやれる？
      - https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#invalidating-the-cache
  - 一方でこれでは普通に困ることもあるので、issueを立ててみました
    - TODO issue探してなければ作成
  - 以下はそこで提案してる内容です
    - cache時間を短くすると負荷問題が顕在化する
      - [ ] どのくらいprefetchするか試してみる
    - 暫定的にはprefetchをやめる
    - より根本的には、App Routerがクライアントサイドのcacheをコントロールできるように提供すべき？

## 感想

TBW
