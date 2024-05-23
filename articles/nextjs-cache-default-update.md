---
title: "Next.js braking change of cache"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

Next.js App Routerは巷では難しいと評されることが多々あります。これはReactの新機能であるServer Componentsをはじめとする**Server 1stへのパラダイムシフト**を必要とすること、そして初見殺しな**デフォルトのキャッシュ挙動**に起因していると筆者は考えています。

パラダイムシフトが必要となるServer ComponentsやServer ActionsなどのReactの新機能については、エラーで指摘・修正のヒントが提示されるなど初学者のフォローもしっかり考慮した設計がなされてたり、多くのドキュメントや記事が公開されているので、これらについてはReact hooksが登場した時のようにあとは時間の問題なのかなとも感じています。

一方でキャッシュについては、デフォルトで積極的かつ何層にも分けてキャッシュされる上、「意図せずキャッシュされてる状態」は当然エラーにならず動作してしまうため、初学者にとってNext.jsのキャッシュはつまづきやすいポイントだと筆者は考えています。筆者は特にクライアントサイドのキャッシュである[Router Cache](https://nextjs.org/docs/app/building-your-application/caching#router-cache)に着目し、その複雑さや問題点について過去記事にしたりしました。

https://zenn.dev/akfm/articles/next-app-router-client-cache

上記記事執筆時点では、Router Cacheは最低でも30sは利用されてしまうことを筆者は問題視していたのですが、その後[`experimental.staleTime`](https://nextjs.org/docs/app/api-reference/next-config-js/staleTimes)が導入されてRouter Cacheの寿命を設定できるようになり、状況は大きく改善されました。

そしてここに来てさらに、v15の更新でRouter Cacheや[Data Cache](https://nextjs.org/docs/app/building-your-application/caching)のdefault設定が変更されることが発表されました。本稿はv15で行われるキャッシュ周りの破壊的変更と、その背景や[PPR](https://nextjs.org/docs/app/api-reference/next-config-js/partial-prerendering)との関係について解説します。

## v15の破壊的変更概要

Next.jsコアチームのメンバーである[Jimmy Lai氏](https://x.com/feedthejim)によって、Next.js@15で変更される内容が公表されました。

https://x.com/feedthejim/status/1792969159321723244

> ◆ no more fetch caching by default ✅(fetchをdefaultでキャッシュすることを廃止)
◆ no more client caching by default ✅(クライアントサイドでdefaultでキャッシュすることを廃止)
◆ no more static GET routes by default ✅(静的GET routeをdefaultでキャッシュすることを廃止 )

これにより、**Data CacheとRouter Cacheがdefaultで無効化**されることになります。その他、破壊的変更ではなく機能追加として以下も発表されました。

https://x.com/feedthejim/status/1792969608489738554

> ◆ incremental PPR migration support(インクリメンタルなPPRマイグレーションをサポート)
◆ next/after, our own little version of waitUntil(`next/after`の追加)
◆ the experimental React Compiler support(React Compiler(別名React Forget)のexperimentalサポート)

TBW: [SHIP](https://vercel.com/ship)で発表された内容を反映

## キャッシュ設定の破壊的変更

### v14以前の初期値

### v15以降の初期値

## なぜこのタイミングで変更されたのか

https://x.com/koichik/status/1793086908299653452

## v15以降でのNext.jsの設計思想

https://x.com/koichik/status/1793092931542487535

## 感想
