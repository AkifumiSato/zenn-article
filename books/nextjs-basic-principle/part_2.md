---
title: "第2部 キャッシュ"
---

App Routerには4層のキャッシュが存在し、デフォルトで積極的に利用されます。以下は[公式の表](https://nextjs.org/docs/app/building-your-application/caching#overview)を翻訳したものです。

| Mechanism               | What                                      | Where  | Purpose                                  | Duration                 |
| ----------------------- | ----------------------------------------- | ------ | ---------------------------------------- | ------------------------ |
| **Request Memoization** | 関数の戻り値                              | Server | React Component treeにおけるdataの再利用 | リクエストごと           |
| **Data Cache**          | APIレスポンスやデータベースアクセスの結果 | Server | ユーザーやデプロイをまたぐデータの再利用 | 永続化 (revalidate可)    |
| **Full Route Cache**    | HTMLやRSC payload                         | Server | レンダリングコストやパフォーマンスの向上 | 永続化 (revalidate可)    |
| **Router Cache**        | RSC Payload                               | Client | ナビゲーションごとのリクエスト削減       | ユーザーセッション・時間 |

すでに[Request Memoization](part_1_request_memoization)については第1部で解説しましたが、これはユーザーリクエスト単位と非常に短い期間で利用されるキャッシュであり、これが問題になることはほとんどないと考えられます。一方他の3つについてはもっと長い期間利用されるため、開発者が意図してコントロールしなければ予期せぬキャッシュによるバグに繋がります。そのため、App Routerを利用する開発者にとってこれらの理解は非常に重要です。

第2部ではApp Routerのキャッシュにまつわる考え方や仕様、コントロールの方法などを解説します。
