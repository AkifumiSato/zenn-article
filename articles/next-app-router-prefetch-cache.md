---
title: "Next.js App Router 遷移の仕組みと実装 - 知られざるprefetch cacheの仕様編"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

## 構成

afddb6ebdade616cdd7780273be4cd28d4509890

- 前回App Routerの遷移周りの実装を研究した記事を投稿しました
  - https://zenn.dev/akfm/articles/next-app-router-navigation
  - 今回はこれの続編として、App Routerのprefetch cacheについてまとめようと思います
- app routerにおけるキャッシュは以下のような分類がある
  - deduping
  - fetch cache
  - CDN cache
  - client router's cache
- 本稿は3つのキャッシュについては触れませんが、fetchのキャッシュ周りの設定はdynamic functionsの有無などによってデフォルトが変わりうるなど、それぞれ複雑さを伴います
  - この辺は他の記事参照
    - https://zenn.dev/cybozu_frontend/articles/next-caching-dedupe
- 一方でサーバーやCDN側のキャッシュとは別に、App Routerにはクライアントサイドにインメモリなキャッシュがある
- 本稿はこれを詳解した記事になります
  - App Routerはインメモリにprefetchをキャッシュに格納する
    - 前回の記事でも説明したように、App Routerは内部的に`useReducer`ベースで作成したStateをRedux Devtoolsに繋げている
    - `prefetchCache: Map<string, PrefetchCacheEntry>`の`data`にprefetchのキャッシュを格納している
    - https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/router-reducer-types.ts#L209-L215
  - 遷移時はこのキャッシュからRSCを取り出してレンダリングする
    - https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/reducers/navigate-reducer.ts#L229
    - ちなみに取り出すのはPromiseを拡張してやってる
      - https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/create-record-from-thenable.ts
      - https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/router-reducer-types.ts#L165-L169
  - prefetchには'auto'/'full'/'temporary'の3種類がある
    - https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/router-reducer-types.ts#L165-L169
    - 遷移時にprefetchがない時には'temporary'なprefetchが作成される
      - https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/reducers/navigate-reducer.ts#L204-L217
  - つまり、内部的には`prefetchCache`はprefetchを無効にしてても遷移時に必要になるので利用されるような作りになってる
  - このキャッシュは条件によるが、30s~5mほど有効になる
    - 該当のリンクのprefetchを無効にした場合などは30s
      - https://github.com/vercel/next.js/blob/afddb6ebdade616cdd7780273be4cd28d4509890/packages/next/src/client/components/router-reducer/get-prefetch-cache-entry-status.ts#L18-L23
    - prefetchキャッシュは5分以内なら再利用される。prefetch cacheのstatusは以下
      - fresh: 新しいキャッシュ
        - prefetchから30s以内
      - reusable: 再利用可能なキャッシュ
        - lastUsedから30s以内
      - stale: ちょっと古いキャッシュ
        - prefetchから5m以内
      - expired: 破棄されるべきキャッシュ
        - prefetchから5m以上
    - flightにレンダリングされたコンポーネントを含まず、staleなときはprefetchを再度行うっぽい
      - https://github.com/vercel/next.js/blob/cad6d3aa2208e82c26eee7ff81a0a47aeec39968/packages/next/src/client/components/router-reducer/apply-flight-data.ts#L15
      - よくわからん...色々試したけど、実際なさそうな気がする
        - 一応これが起点っぽい
        - https://github.com/vercel/next.js/pull/44502
    - この辺の話は仕様としては記載なさそう
      - https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#prefetching
- とはいえこれでは`no-store`のように常に更新を促したい時にうまくいかない
- ではどうするべきか
  - 現状特にやりようはない
  - 一方でこれでは普通に困ることもあるので、issueを立ててみました
    - TODO issue探してなければ作成
  - 以下はそこで提案してる内容です
    - キャッシュ時間を短くすると負荷問題が顕在化する
      - [ ] どのくらいprefetchするか試してみる
    - 暫定的にはprefetchをやめる
    - より根本的には、App Routerがクライアントサイドのキャッシュをコントロールできるように提供すべき？

