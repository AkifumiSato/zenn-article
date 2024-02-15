---
title: "Custom Next.js Cache Handler
 - Vercel以外でのNext.jsキャッシュ活用"
emoji: "💸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

Next.jsアプリケーションのデプロイ先として[Vercel](https://vercel.com/)はとても利便性が高く、優れたプラットフォームです。一方で、インフラ的な都合やコスト的な都合で[Self-hosting](https://nextjs.org/docs/app/building-your-application/deploying#self-hosting)、つまりVercel以外を利用したいケースはそれなりに多いのではないでしょうか。実際、筆者の周りではSelf-hostingでNext.jsを利用しているケースは多く見受けられます。Next.jsはVercel非依存なOSSと銘打ってますが、実際にはNext.jsに必要なインフラ仕様をVercelが隠蔽しているため、Self-hostingはNext.jsの理解とインフラ理解が求められる高いハードルが存在します。昨今、[App Router](https://nextjs.org/docs/routing/introduction)の登場とその強力なキャッシュ戦略により、Vercel以外でNext.jsを扱うことはより難しくなってきました。

一方で、Self-hosting向けのドキュメントや対応は少しづつであるが取り組みがなされています。今回はその一貫で出てきた、**Custom Next.js Cache Handler**を利用してNext.jsのキャッシュをRedisに保存する方法について紹介します。

## Next.jsのキャッシュ

Next.jsにはいくつかのキャッシュが存在し、App Routerにおいては4種類ものキャッシュがあります。

https://nextjs.org/docs/app/building-your-application/caching

特にリクエストを跨いでサーバー側で共有されるキャッシュとして、Pages Routerでは[ISR](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)のキャッシュ、App Routerではデフォルトで有効な[Full Route Cache](https://nextjs.org/docs/app/building-your-application/caching#full-route-cache)や[Data Cache](https://nextjs.org/docs/app/building-your-application/caching#data-cache)が挙げられます。これらは**デフォルトではファイルキャッシュ**となっています。

Self-hostingのインフラ構成は通常複数サーバーやサーバーレスであろうことを想定すると、ファイルキャッシュは少々勝手が悪いものと言えます。AWSでは[EFS](https://aws.amazon.com/jp/efs/)などを用いて複数インスタンス間でファイル共有する手段など考えられますが、CDN側で[オリジンシールド](https://dev.classmethod.jp/articles/amazon-cloudfront-support-cache-layer-origin-shield/)を有効にしていないとファイルI/Oの競合状態が発生しうるなど注意すべき点があり、Next.jsとインフラ構成について高い理解が必要となります。

これらの代替策としてキャッシュ永続用のRedisなどを用意することが考えられますが、残念ながらNext.jsにはこれまでキャッシュ永続化をカスタマイズできるオプションなどは存在しませんでした。

## Self-hostingサポート

しかしApp Router登場以降、Self-hosting周りのフィードバックが多数あったようで、キャッシュ永続化先をカスタム実装できる**Custom Next.js Cache Handler**という機能が実装されました。

https://nextjs.org/docs/app/api-reference/next-config-js/incrementalCacheHandlerPath

これにより、ファイルキャッシュではなくRedisでキャッシュ永続化ができるようになったため、Self-hostingでのApp RouterやISR利用についてより柔軟な対応が可能となりました。

また、これとほぼ同時にドキュメントの改善も行われ、従来より公式ドキュメントから情報を得られるようになりました。Self-hostingサポートは大きく進展したと言えると思います。

https://github.com/vercel/next.js/pull/58027

## Custom Next.js Cache Handler

さて、Self-hostingでは必須に近いこのCustom Cache Handlerですが、まずこの機能を利用するには`next.conifg.js`で`cacheMaxMemorySize: 0`を指定して、Next.jsのインメモリなキャッシュを無効にする必要があります。

```js
module.exports = {
  cacheMaxMemorySize: 0, // disable default in-memory caching
}
```

Next.jsにはファイルキャッシュから読み取った値をさらにインメモリに保存する内部キャッシュが存在し、上記はこれを無効にするオプションです。これをしないとせっかくファイルキャッシュをやめてもキャッシュ不整合が発生する余地が生まれてしまうので、指定は必須です。

キャッシュのハンドリング処理自体はインターフェース仕様に基づいて実装することでカスタマイズできます。現在は3つのメソッド（`get`/`set`/`revalidateTag`）を実装することで、キャッシュの読み書きなどの実装が可能です。

https://nextjs.org/docs/app/api-reference/next-config-js/incrementalCacheHandlerPath#api-reference

:::message
メソッドに`revalidatePath`がないのは、内部的に`revalidateTag`を呼び出しており実質的にwrapperであるためです。
:::

しかし、これらを自前で実装するのはRedisやキャッシュ周りの操作知識も必要となり、また大抵の場合似通ってくることから少々面倒な作業です。そのため公式のexamplesでは、`neshca`というライブラリを利用した実装例が提供されています。

## `neshca`

`neshca`というライブラリは、**Next.js Shared Cache**の略です。

https://caching-tools.github.io/next-shared-cache

Vercelとの直接的な関係は確認できてないので、おそらく有志による開発だと思われます。これを使うことで、Redisに対するキャッシュの読み書きがかなり簡単に実装できます。

Next.jsの公式examplesである[Custom Cache Handlerの実装例](https://github.com/vercel/next.js/tree/10599a4e1eb442306def0de981cbc96b83e6f6f0/examples/cache-handler-redis)から、以下のような実装でRedisにキャッシュを永続化することが可能です。

```js
// next.config.js
module.exports = {
  cacheHandler: require.resolve('./cache-handler.js'),
  cacheMaxMemorySize: 0, // disable default in-memory caching
}

// cache-handler.js
const { IncrementalCache } = require("@neshca/cache-handler");
const createRedisCache = require("@neshca/cache-handler/redis-stack").default;
const createLruCache = require("@neshca/cache-handler/local-lru").default;
const { createClient } = require("redis");

const client = createClient({
  url: process.env.REDIS_URL ?? "redis://localhost:6379",
});

client.on("error", (error) => {
  console.error("Redis error:", error.message);
});

IncrementalCache.onCreation(async () => {
  // read more about TTL limitations https://caching-tools.github.io/next-shared-cache/configuration/ttl
  function useTtl(maxAge) {
    const evictionAge = maxAge * 1.5;

    return evictionAge;
  }

  let redisCache;

  if (process.env.REDIS_AVAILABLE) {
    await client.connect();

    redisCache = await createRedisCache({
      client,
      useTtl,
    });
  }

  const localCache = createLruCache({
    useTtl,
  });

  return {
    cache: [redisCache, localCache],
    // read more about useFileSystem limitations https://caching-tools.github.io/next-shared-cache/configuration/use-file-system
    useFileSystem: false,
  };
});

module.exports = IncrementalCache
```

あとは実際のRedisインスタンスをDockerなどで立てて`REDIS_URL`を指定してあげれば、ローカル環境でもキャッシュをRedisに保存する様になります。筆者としては上記に加え、[キャッシュのkeyにgit hashをprefixにするなどの設定](https://caching-tools.github.io/next-shared-cache/configuration/build-id-as-prefix-key)を追加したいところです。

```js
const redisCache = await createRedisCache({
  client,
  keyPrefix: process.env.GIT_HASH,
});
```

## まとめ

`neshaca`やCustom Next.js Cache Handlerはまだ登場したばかりのため、プロダクションでの実運用においてどういう問題が起きるかなどについては未知数です。一方、これまでSelf-hosting向けのサポートは手薄く感じていたので、こういう機能が出てきたことは大きな進展とも考えています。

実際、これはPaaSのVercelを売るには不利な機能なはずです。それでもこういう機能が出てきたのは、App Routerの普及にはSelf-hostingのサポート需要が無視できないものだったということなのかもしれません。今後Self-hosting向けのNext.jsの機能開発がより進展することを期待したいところです。
