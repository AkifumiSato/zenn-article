---
title: "第3部 キャッシュ"
---

App Routerには4層のキャッシュが存在し、デフォルトで積極的に利用されます。以下は[公式の表](https://nextjs.org/docs/app/building-your-application/caching#overview)を翻訳したものです。

| Mechanism               | What              | Where  | Purpose                                    | Duration                                |
| ----------------------- | ----------------- | ------ | ------------------------------------------ | --------------------------------------- |
| **Request Memoization** | APIレスポンスなど | Server | React Component treeにおけるデータの再利用 | リクエストごと                          |
| **Data Cache**          | APIレスポンスなど | Server | ユーザーやデプロイをまたぐデータの再利用   | 永続的 (revalidate可)                   |
| **Full Route Cache**    | HTMLやRSC payload | Server | レンダリングコストの最適化                 | 永続的 (revalidate可)                   |
| **Router Cache**        | RSC Payload       | Client | ナビゲーションごとのリクエスト削減         | ユーザーセッション・時間 (revalidate可) |

:::message alert
Next.jsの`v15.0.0-rc.0`では[デフォルトのキャッシュ設定の見直し](https://nextjs.org/blog/next-15-rc#caching-updates)が行われ、従来ほど積極的なキャッシュは行われなくなりましたが、本書執筆時点でv15はまだRC版のため、本書の内容はv14の内容に基づいています。
:::

すでにRequest Memoizationについては[_第1部 データフェッチ_](part_1)で解説しましたが、これはNext.jsサーバーに対するリクエスト単位の非常に短い期間でのみ利用されるキャッシュであり、これが問題になることはほとんどないと考えられます。一方他の3つについてはもっと長い期間および広いスコープで利用されるため、開発者が意図してコントロールしなければ予期せぬキャッシュによるバグに繋がりかねません。そのため、App Routerを利用する開発者にとってこれらの理解は非常に重要です。

第3部ではApp Routerにおけるキャッシュの考え方や仕様、コントロールの方法などを解説します。
