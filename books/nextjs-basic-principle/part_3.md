---
title: "第3部 キャッシュ"
---

Next.jsには4層のキャッシュが存在し、デフォルトで積極的に利用されます。以下は[公式の表↗︎](https://nextjs.org/docs/app/guides/caching#overview)を翻訳したものです。

| Mechanism               | What              | Where  | Purpose                                    | Duration                                |
| ----------------------- | ----------------- | ------ | ------------------------------------------ | --------------------------------------- |
| **Request Memoization** | APIレスポンスなど | Server | React Component treeにおけるデータの再利用 | リクエストごと                          |
| **Data Cache**          | APIレスポンスなど | Server | ユーザーやデプロイをまたぐデータの再利用   | 永続的 (revalidate可)                   |
| **Full Route Cache**    | HTMLやRSC payload | Server | レンダリングコストの最適化                 | 永続的 (revalidate可)                   |
| **Router Cache**        | RSC Payload       | Client | ナビゲーションごとのリクエスト削減         | ユーザーセッション・時間 (revalidate可) |

:::message
Next.js v15で[キャッシュのデフォルト設定の見直し↗︎](https://nextjs.org/blog/next-15-rc#caching-updates)が行われ、従来ほど積極的なキャッシュは行われなくなりました。
現在実験的機能である[`cacheComponents`↗︎](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents)フラグでは、さらにサーバーサイドキャッシュはデフォルトで無効化され、`"use cache"`でキャッシュをオプトインする設計に変更することが検討されています。
:::

すでに[Request Memoization](part_1_request_memoization)については解説しましたが、これはNext.jsサーバーに対するリクエスト単位という非常に短い期間でのみ利用されるキャッシュであり、これが問題になることはほとんどないと考えられます。一方他の3つについてはもっと長い期間および広いスコープで利用されるため、開発者が意図してコントロールしなければ予期せぬキャッシュによるバグに繋がりかねません。そのため、Next.jsを利用する開発者にとってこれらの理解は非常に重要です。

第3部ではNext.jsにおけるキャッシュの考え方や仕様、コントロールの方法などを解説します。
