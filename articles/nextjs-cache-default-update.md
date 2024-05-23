---
title: "Next.js braking change - disable router/fetch cache by default"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: true
---

Next.js App Routerは巷では難しいと評されることが多々あります。これはReactの新機能であるServer Componentsをはじめとする**Server 1stとも言えるパラダイムシフト**を必要とすること、そして初見殺しな**デフォルトのキャッシュ挙動**に起因していると筆者は考えています。

パラダイムシフトが必要となるServer ComponentsやServer ActionsなどのReactの新機能については、エラーで指摘・修正のヒントが提示されるなどの初学者のフォローもしっかり考慮した設計がなされてたり、多くのドキュメントや記事が公開されているので、これらについてはhooksが登場した時のようにあとは世の中に理解が広まるまでの時間の問題なのかなとも感じています。

一方でキャッシュについては、デフォルトで積極的かつ何層にも分けてキャッシュされる上、「意図せずキャッシュされてる状態」は当然エラーにならず動作してしまうため、初学者にとってNext.jsのキャッシュはつまづきやすいポイントだと筆者は感じています。筆者は特に、クライアントサイドのキャッシュである[Router Cache](https://nextjs.org/docs/app/building-your-application/caching#router-cache)に着目しその複雑さや問題点について過去に記事にしたこともありました。

https://zenn.dev/akfm/articles/next-app-router-client-cache

上記執筆時点では、Router Cacheは最低でも30sは利用されてしまうことを筆者は問題視していたのですが、その後`experimental.staleTime`が導入されてRouter Cacheの有効期限を設定できるようになり、状況は大きく改善されました。

https://nextjs.org/docs/app/api-reference/next-config-js/staleTimes

そしてここに来てさらに、v15でRouter Cacheや[Data Cache](https://nextjs.org/docs/app/building-your-application/caching)のデフォルト設定が変更されることが発表されました。これは非常に大きな変更だと筆者は捉えています。

本稿ではv15で行われるキャッシュ周りの破壊的変更と、その背景や[PPR](https://nextjs.org/docs/app/api-reference/next-config-js/partial-prerendering)との関係について解説します。

## v15の破壊的変更概要

Next.jsコアチームのメンバーである[Jimmy Lai氏](https://twitter.com/feedthejim)によって、Next.jsのv15で変更される内容が公表されました。

https://twitter.com/feedthejim/status/1792969159321723244

> ◆ no more fetch caching by default ✅(fetchをデフォルトキャッシュすることを廃止)
> ◆ no more client caching by default ✅(クライアントサイドでデフォルトキャッシュすることを廃止)
> ◆ no more static GET routes by default ✅(GET routeをデフォルト静的化することを廃止)

これにより、**Data CacheとRouter Cacheがデフォルトで無効化**されることになります。その他、機能追加として以下も発表されました。

https://twitter.com/feedthejim/status/1792969608489738554

> ◆ incremental PPR migration support(インクリメンタルなPPRマイグレーションをサポート)
> ◆ next/after, our own little version of waitUntil(`next/after`の追加)
> ◆ the experimental React Compiler support(React Compiler(別名React Forget)のexperimentalサポート)

これらに対するupgradeガイドは本稿執筆時点では公式ドキュメント上にまだ公開されてないものの、リポジトリ上ではマージされていることが確認できました。

https://github.com/vercel/next.js/blob/93c861d67bfb88109ee3bb7ddc9b8801f0c07bba/docs/02-app/01-building-your-application/11-upgrading/02-version-15.mdx

## キャッシュ設定の破壊的変更

前述の通り、v15でData CacheとRouter Cacheはデフォルトで無効化されます。簡単におさらいするとData Cacheは`fetch`はじめサーバー側でのデータアクセス時に保持されるデータそのもののキャッシュで、Router Cacheはクライアントサイドに保持されるRSC Payloadのキャッシュです。

詳しくは以下をご参照ください。

https://nextjs.org/docs/app/building-your-application/caching

### Data Cacheの無効化

v14以前は、`fetch`を使ったデータ取得はデフォルトで無期限にキャッシュされていました。これはNext.jsが拡張した`fetch`のオプションである[cache](https://nextjs.org/docs/app/building-your-application/caching#fetch-optionscache)や[next.revalidate](https://nextjs.org/docs/app/api-reference/functions/fetch#optionsnextrevalidate)によって変更が可能でした。

```ts
// fetch時に`cache: 'no-store'`を指定してopt-out
fetch(`https://...`, { cache: 'no-store' })

// fetch時に`next: { revalidate: 0 }`を指定してopt-out
fetch('https://...', { next: { revalidate: 0 } })

// fetch時に`next: { revalidate: 3600 }`を指定して有効期限を設定
fetch('https://...', { next: { revalidate: 3600 } })
```

v15以降、Data Cacheはデフォルトで無効化されるので、上記方法によってData Cacheをopt-outする必要はなくなりました。opt-inする場合には、以下のように実装する必要があります。

```ts
// v14以前同様、無期限キャッシュをopt-in
fetch(`https://...`, { cache: 'force-store' })

// fetch時に`next: { revalidate: 3600 }`を指定して有効期限を設定
fetch('https://...', { next: { revalidate: 3600 } })
```

:::message
Route Segment Configの[fetchCache](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#fetchcache)を設定すれば、fetchのオプションのデフォルトを`{ cache: 'force-store' }`にすることができます。ただしこれはドキュメントの通り、高度なオプションです。設定する場合は慎重に検討することをお勧めします。
:::

### Router Cacheの無効化

Router Cacheのデフォルト有効期限はいくつかの条件によって決定されるのですが、ほとんどの場合は[dynamic rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)かどうかによって決定されます。

:::message
実際には`Link`コンポーネントの`prefetch`propsや、`router.prefetch()`によって有効期限が変わることがあります。興味のある方は筆者の[過去の記事](https://zenn.dev/akfm/articles/next-app-router-client-cache#client-side-cache%E3%81%AE%E7%A8%AE%E5%88%A5)をご参照ください。
:::

v14以前はstatic renderingなら5m、dynamic renderingなら30sがデフォルトで設定されていました。v15以降、**dynamic renderingのデフォルトが0sに変更**されます。staticには変更ありません。

dynamic renderingはRoute Segment Configの[dynamic](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#dynamic)の設定や[dynamic functions](https://nextjs.org/docs/app/building-your-application/routing/route-handlers#dynamic-functions)の利用有無によって決定されるので、これらを利用していないstatic renderingなページにおいてはRouter Cacheがデフォルトでは無効化されないということです。

| rendering | e.g.           | v14     | v15    |
|-----------|----------------|---------|--------|
| static    | 静的ページ、ブログ記事ページ | 5m      | 5m     |
| dynamic   | ユーザーのマイページ     | **30s** | **0s** |

これにより、ブログ記事ページをrevalidateしたのに他のユーザーにはRouter Cacheが残ってて古い情報が見えてしまうなどのケースが想定されます。これを制御したい場合、前述の`staleTime`を設定する必要があります。

https://nextjs.org/docs/app/api-reference/next-config-js/staleTimes

とはいえ個人的な意見としては、static renderingはデフォルトで強くキャッシュしても違和感はないので、デフォルト設定としてはこれは悪くない判断じゃないかなと思っています。

## なぜこのタイミングで変更されたのか

実際に使ってみないとわからない部分もあるかもしれませんが、これらの情報を眺める限りでは基本的に初見殺しだったキャッシュ周りが改善される良いbreaking changeじゃないかなと筆者は考えています。

しかし、キャッシュ周りについては以前から[Discussionで強くフィードバック](https://github.com/vercel/next.js/discussions/54075)されてたり要望は多かったのですが、なぜこのタイミングでの変更となったのでしょう？これについてもコアチームのJimmy Lai氏がツイートで説明しています。(より詳細な説明は近日中にブログで公開されるとのことです。)

https://twitter.com/feedthejim/status/1792973728512426304

上記ツイートは要約すると、以下のような理由が挙げられています。

- 2023/11時点でキャッシュのデフォルトを変更したいと考えていた
- しかし、この変更を実施するにはトレードオフを正確に見極める必要があった
- 今年の最大の焦点はPPRの完成であり、PPRはキャッシュへの依存があったためPPRの設計が固めることの方が先だった
- すでにPPRの設計は議論の余地がないところまで進んだため、キャッシュのデフォルト変更を実施することができた

コアメンバーの説明は上記のようなものですが、これに加えkoichikさんの考察が納得行くものだったので引用します。

https://twitter.com/koichik/status/1793086908299653452

PPRによって動的なfetchがあっても静的化できるようになったので、Data Cacheをデフォルトで有効にしておくモチベーションがなくなったのです。デフォルトにおいては、データ単位のキャッシュに代わってhtmlやRSC Payload単位でのキャッシュされる範囲が広がったとイメージするとわかりやすいかもしれません。

## v15以降でのNext.jsの設計思想

もう1つkoichikさんのツイートで重要だと思ったのが、Next.jsの設計思想についての変革です。

https://twitter.com/koichik/status/1793092931542487535

Next.js v15以降はページ単位の静的・動的という思想ではなく、ページやSuspenseによって隔てられた境界ごとに静的・動的が決定できるようになります。開発者としては「必要な部分だけを動的にすることができる」と捉えることもできるので、レンダリング設計としてはシンプルでわかりやすい気もします。

## 感想

Next.jsは定期的に大きな概念の追加やアップデートが行われます。巷で「進化が早すぎる」「too much」などと評されるのはそれが原因なのかもしれません。一方でその進化はユーザーの体験へと繋がるものが多く、重視すべき点だと筆者は考えています。

PPRは1つの新たな概念なので、我々がそれを理解し使いこなすには学習コストが伴います。正直筆者自身、まだPPRについての理解度や解像度は荒い部分があります。一方でPPR以降の世界では前述の通り「必要な部分だけを動的にすることができる」という世界になることで、レンダリングプロセスの理解容易性とユーザー体験どちらをとっても歓迎すべき変更な気はしてます。パフォーマンス観点はアプリケーション実装者がチューニングしなければならないケースも当然ありますが、フレームワーク側でデフォルトで対処されてると平均値が上がっていくので1Webユーザーとしても嬉しいところです。

PPRが安定化して使える日が近い将来なのだと思うと楽しみですが、フレームワーク内部としては大幅な変更でしょうから、バグや既存影響など出ないかは少々心配なところです。Next.jsの場合v15への更新に限らないですが、アップデートは慎重に行いましょう。
