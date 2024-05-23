---
title: "Next.js braking change - cache by default"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

Next.js App Routerは巷では難しいと評されることが多々あります。これはReactの新機能であるServer Componentsをはじめとする**Server 1stへのパラダイムシフト**を必要とすること、そして初見殺しな**デフォルトのキャッシュ挙動**に起因していると筆者は考えています。

パラダイムシフトが必要となるServer ComponentsやServer ActionsなどのReactの新機能については、エラーで指摘・修正のヒントが提示されるなど初学者のフォローもしっかり考慮した設計がなされてたり、多くのドキュメントや記事が公開されているので、これらについてはhooksが登場した時のようにあとは理解が世の中に広まるまでの時間の問題なのかなとも感じています。

一方でキャッシュについては、デフォルトで積極的かつ何層にも分けてキャッシュされる上、「意図せずキャッシュされてる状態」は当然エラーにならず動作してしまうため、初学者にとってNext.jsのキャッシュはつまづきやすいポイントだと筆者は考えています。筆者は特にクライアントサイドのキャッシュである[Router Cache](https://nextjs.org/docs/app/building-your-application/caching#router-cache)に着目し、その複雑さや問題点について過去記事にしたりしました。

https://zenn.dev/akfm/articles/next-app-router-client-cache

上記執筆時点では、Router Cacheは最低でも30sは利用されてしまうことを筆者は問題視していたのですが、その後`experimental.staleTime`が導入されてRouter Cacheの寿命を設定できるようになり、状況は大きく改善されました。

https://nextjs.org/docs/app/api-reference/next-config-js/staleTimes

そしてここに来てさらに、v15でRouter Cacheや[Data Cache](https://nextjs.org/docs/app/building-your-application/caching)のdefault設定が変更されることが発表されました。本稿はv15で行われるキャッシュ周りの破壊的変更と、その背景や[PPR](https://nextjs.org/docs/app/api-reference/next-config-js/partial-prerendering)との関係について解説します。

## v15の破壊的変更概要

Next.jsコアチームのメンバーである[Jimmy Lai氏](https://twitter.com/feedthejim)によって、Next.js@15で変更される内容が公表されました。

https://twitter.com/feedthejim/status/1792969159321723244

> ◆ no more fetch caching by default ✅(fetchをdefaultでキャッシュすることを廃止)
> ◆ no more client caching by default ✅(クライアントサイドでdefaultでキャッシュすることを廃止)
> ◆ no more static GET routes by default ✅(GET routeをdefaultで静的化することを廃止)

これにより、**Data CacheとRouter Cacheがdefaultで無効化**されることになります。その他、破壊的変更ではなく機能追加として以下も発表されました。

https://twitter.com/feedthejim/status/1792969608489738554

> ◆ incremental PPR migration support(インクリメンタルなPPRマイグレーションをサポート)
> ◆ next/after, our own little version of waitUntil(`next/after`の追加)
> ◆ the experimental React Compiler support(React Compiler(別名React Forget)のexperimentalサポート)

TBW: [SHIP](https://vercel.com/ship)で発表された内容を反映

## キャッシュ設定の破壊的変更

前述の通り、v15でData CacheとRouter Cacheはdefaultで無効化されます。簡単におさらいするとData Cacheは`fetch`はじめサーバー側でのデータアクセス時に保持されるデータそのもののキャッシュで、Router Cacheはクライアントサイドに保持されるRSC Payloadのキャッシュです。

詳しくは以下をご参照ください。

https://nextjs.org/docs/app/building-your-application/caching

### Data Cacheの無効化

v14以前は、`fetch`を使ったデータ取得はdefaultで無期限にキャッシュされていました。これはNext.jsが拡張した`fetch`のオプションである[cache](https://nextjs.org/docs/app/building-your-application/caching#fetch-optionscache)や[next.revalidate](https://nextjs.org/docs/app/api-reference/functions/fetch#optionsnextrevalidate)によって変更が可能でした。

```ts
// fetch時に`cache: 'no-store'`を指定してopt-out
fetch(`https://...`, { cache: 'no-store' })

// fetch時に`next: { revalidate: 0 }`を指定してopt-out
fetch('https://...', { next: { revalidate: 0 } })

// fetch時に`next: { revalidate: 3600 }`を指定して有効期限を設定
fetch('https://...', { next: { revalidate: 3600 } })
```

v15以降、Data Cacheはデフォルトで無効化されるので、上記方法によってData Cacheをopt-outする必要はなくなりました。

### Router Cacheの無効化

Router Cacheのdefault有効期限はいくつかの条件によって決定されるのですが、ほとんどの場合は[dynamic rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)かどうかによって決定されます。

:::message
実際には`Link`コンポーネントの`prefetch`propsや、`router.prefetch()`によって有効期限が変わることがあります。興味のある方は筆者の[過去の記事](https://zenn.dev/akfm/articles/next-app-router-client-cache#client-side-cache%E3%81%AE%E7%A8%AE%E5%88%A5)をご参照ください。
:::

v14以前は、static renderingなら5m、dynamic renderingなら30sがdefaultで設定されていました。v15以降、**dynamic renderingのdefaultが0sに変更**されます。staticには変更ありません。

dynamic renderingはRoute Segment Configの[dynamic](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#dynamic)の設定や[dynamic functions](https://nextjs.org/docs/app/building-your-application/routing/route-handlers#dynamic-functions)の利用有無によって決定されるので、これらを利用していないstatic renderingなページにおいてはRouter Cacheがdefaultでは無効化されないということです。

| rendering | e.g.           | v14 | v15     |
|-----------|----------------|-----|---------|
| static    | 静的ページ、ブログ記事ページ | 5m  | **30s** |
| dynamic   | ユーザーのマイページ     | 5m  | **0s**  |

ブログ記事ページをrevalidateしたのに、他のユーザーにはRouter Cacheが残ってて古い情報が見えてしまうなどのケースが想定されます。これを制御したい場合、前述の`staleTime`を設定する必要があります。

https://nextjs.org/docs/app/api-reference/next-config-js/staleTimes

static renderingはdefaultで強くキャッシュしても個人的には違和感はないので、個人的にはいい判断じゃないかなと思っています。

## なぜこのタイミングで変更されたのか

実際に使ってみないとわからない部分もあるかもしれませんが、これらの情報を眺める限りでは、基本的に初見殺しだったキャッシュ周りが改善される良いbreaking changeじゃないかなと筆者は考えています。

しかし、キャッシュ周りについては以前から[Discussionで強くフィードバック](https://github.com/vercel/next.js/discussions/54075)されてたり要望は多かったのですが、なぜこのタイミングでの変更となったのでしょう？

これについてもコアチームのJimmy Lai氏がツイートで説明しています。

https://twitter.com/feedthejim/status/1792973728512426304

より詳細な説明は近日中にブログで公開されるとのことです。

Todo: ツイートの概要説明から

https://twitter.com/koichik/status/1793086908299653452

## v15以降でのNext.jsの設計思想

https://twitter.com/koichik/status/1793092931542487535

## 感想
