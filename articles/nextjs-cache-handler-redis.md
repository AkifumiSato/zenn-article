---
title: "Custom Next.js Cache Handler
 - Vercel以外でのNext.jsキャッシュ活用"
emoji: "💸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

Next.jsアプリケーションのデプロイ先として[Vercel](https://vercel.com/)はとても利便性が高く、優れたプラットフォームです。一方で、インフラ的な都合やコスト的な都合で[Self-hosting](https://nextjs.org/docs/app/building-your-application/deploying#self-hosting)、つまりVercel以外を利用したいケースはそれなりに多いのではないでしょうか。実際、筆者の周りではSelf-hostingでNext.jsを利用しているケースは多く見受けられます。Next.jsはVercel非依存なOSSと銘打ってますが、実際にはVercelが隠蔽してるNext.jsが求めるインフラ仕様を実現するのは高いハードルです。昨今、[App Router](https://nextjs.org/docs/routing/introduction)の登場とその強力なキャッシュ戦略により、よりVercel以外でNext.jsを扱うことは難しくなってきました。一方で、Self-hosting向けのドキュメントや対応は少しづつであるが取り組みがなされています。今回はその一貫で出てきた、`cache-handler`を利用してNext.jsのキャッシュをRedisに保存する方法について紹介します。

## Next.jsのキャッシュ

Next.jsにはいくつかのキャッシュが存在し、App Routerにおいては4種類ものキャッシュがあります。

https://nextjs.org/docs/app/building-your-application/caching

特にプロセスを跨いで共有されるキャッシュとして、Pages Routerでは[ISR](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)有効時、App Routerではデフォルトで有効な[Full Route Cache](https://nextjs.org/docs/app/building-your-application/caching#full-route-cache)や[Data Cache](https://nextjs.org/docs/app/building-your-application/caching#data-cache)が挙げられます。これらのキャッシュは、**デフォルトではファイルキャッシュ**となっています。

Self-hostingのインフラ構成で複数サーバーやサーバーレスを想定すると、ファイルキャッシュは少々勝手が悪いものと言えます。AWSでは[EFS](https://aws.amazon.com/jp/efs/)などを用いてファイル共有する手段などが考えられますが、ファイルI/Oの競合なども考えられるためCDN側でオリジンシールドの設定が必要など注意すべき点があり、Next.jsとインフラ構成について高い理解が必要となります。

これらの代替策としてキャッシュ永続用のRedisなどを用意することが考えられますが、残念ながらNext.jsのキャッシュ永続化をカスタマイズするようなオプションは存在しませんでした。

## Self-hostingサポート

しかしApp Router登場以降、Self-hosting周りのフィードバックが多数あったようで、キャッシュ永続化先をカスタム実装できる[Custom Next.js Cache Handlerという機能が実装](https://github.com/vercel/next.js/pull/57953)されました。

https://nextjs.org/docs/app/api-reference/next-config-js/incrementalCacheHandlerPath

これにより、ファイルキャッシュではなくRedisにキャッシュを永続化することができるため、App RouterやISR利用時のSelf-hostingについて従来よりも柔軟な対応が可能となりました。

:::message
余談ですが、これとほぼ同時にドキュメントの改善も行われ、従来よりSelf-hostingの手探りな状態から公式ドキュメントから情報を得られるようになりました。Self-hostingサポートは大きく進展したと言えると思います。

https://github.com/vercel/next.js/pull/58027
:::

## Custom Next.js Cache Handler

Self-hostingでは必須に近いこの`Custom Next.js Cache Handler`ですが、まずこの機能を利用するには`next.conifg.js`で`cacheMaxMemorySize: 0`を指定して、Next.jsのインメモリなキャッシュを無効にする必要があります。

```js
module.exports = {
  cacheMaxMemorySize: 0, // disable default in-memory caching
}
```

これはファイルキャッシュから読み取った値をさらにインメモリに保存するNext.jsの内部キャッシュを無効にするオプションです。これをしないとキャッシュとの不整合が発生しうるので指定は必須です。

キャッシュのハンドリング処理自体はAPI仕様に基づいて実装することでカスタマイズできます。現在は3つのメソッド（`get`/`set`/`revalidateTag`）を実装することで、キャッシュの読み書きなどの実装が可能です。

https://nextjs.org/docs/app/api-reference/next-config-js/incrementalCacheHandlerPath#api-reference

`revalidateTag`のみで`revalidatePath`がないのは、`revalidatePath`は内部的に`revalidateTag`を呼び出しており実質的にwrapperであるためです。

しかしこれらを自前で実装するのはRedisや`ioredis`などの知識も必要となり、少々骨が折れる作業です。そのため、公式のexamplesでは、`neshca`というライブラリを利用した実装例が提供されています。

## `neshca`

Next.jsの公式examplesであるCustom Cache Handlerの実装例は以下です。

https://github.com/vercel/next.js/tree/10599a4e1eb442306def0de981cbc96b83e6f6f0/examples/cache-handler-redis

ここで利用されている`neshca`というライブラリは、Next.js Shared Cacheの略です。

https://caching-tools.github.io/next-shared-cache

Vercelとの直接的な関係は確認できてないので、おそらく有志による開発だと思われます。これを使うことで、Redisに対するキャッシュの読み書きがかなり簡単に実装できます。

todo: cache-handler.jsのサンプル

## 構成

- まとめ
  - `neshca`や`Custom Next.js Cache Handler`はまだ登場したばかりのため、プロダクションでの実運用においてどういう問題が起きるかなどについては未知数です
  - 一方、これまでSelf-hosting向けのサポートは手薄く感じていたので、こういう機能が出てきたことは大きな進展とも考えています
  - 実際、これはVercelにとっては不利な機能開発なはずです
  - それでもこういう機能が出てきたのは、App Routerの普及にはSelf-hostingのサポート需要が無視できないものだったということなのかもしれません
  - 今後Self-hosting向けのNext.jsの機能開発がより進展することを期待したいところです

## todo

- 簡単なサンプルを実装する
  - Redis(not stack)で構築する
  - git hashをprefixにする
    - https://caching-tools.github.io/next-shared-cache/configuration/build-id-as-prefix-key
  - TTL設定
