---
title: "Next.js Cache回想"
emoji: "👨‍💻"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
---

:::message
この記事は[JSConf JP 2025で発表した内容](https://jsconf.jp/2025/ja)を、記事として執筆しなおしたものです。
:::

Next.jsのCacheは、パフォーマンス最適化において重要な機能である一方、開発者に多くの混乱をもたらしてきました。特に、デフォルトで有効なCacheや複雑すぎる影響範囲は、開発者にとっては大きな負担となっていました。v16で導入された[Cache Components](https://nextjs.org/docs/app/getting-started/cache-components)は、Next.jsのCacheのメンタルモデルを大きく変更するものであり、これまでの多くの課題を解決しうる根本的なアプローチです。

本稿ではNext.jsにおけるCacheの歴史の振り返ることで、Cache Componentsの背景理解を深めることを目指します。

## Next.jsの歴史

Next.jsには現在、**Pages Router**と**App Router**2つのRouterが同梱されています。2016年に発表された初期のNext.jsが現在Pages Routerと呼ばれるものであり、その後[React Server Components](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)をサポートした新たなRouterとして、App Routerが開発されました。

| 年      | 概要            | 詳細                                |
| ------- | --------------- | ----------------------------------- |
| 2013/05 | Reactが公開     |                                     |
| 2016/10 | Next.jsが公開   |                                     |
| 2018~   | Gatsby.jsの台頭 | SSGが注目される                     |
| 2019/07 | Next.js@v9.0    | dynamicルーティング・Typescript対応 |
| 2020/03 | Next.js@v9.3    | SSG対応                             |
| 2020/07 | Next.js@v9.5    | ISR対応                             |
| 2022/05 | Layout RFC発表  | 後のApp Router                      |
| 2022/10 | Next.js@v13.0   | App Router(Beta)                    |
| 2023/05 | Next.js@v13.4   | App Router(Stable)                  |

Next.jsは従来より、パフォーマンスと開発者体験を重視してきました。SSGが注目されてからは、Next.jsもSSGサポートやISRの発明など、静的化によるパフォーマンス最適化を積極的に行ってきました。

## App Router初期のCache

Pages Routerで培われた静的化によるパフォーマンス最適化思想はApp Routerでも引き継がれており、ページの静的化はCacheの1種として位置付けられ、より高度な最適化のために多層のCacheが導入されました。

以下は[公式の表↗︎](https://nextjs.org/docs/app/guides/caching#overview)を翻訳したものです。

| Mechanism               | What              | Where  | Purpose                                    | Duration                                |
| ----------------------- | ----------------- | ------ | ------------------------------------------ | --------------------------------------- |
| **Request Memoization** | APIレスポンスなど | Server | React Component treeにおけるデータの再利用 | リクエストごと                          |
| **Data Cache**          | APIレスポンスなど | Server | ユーザーやデプロイをまたぐデータの再利用   | 永続的 (revalidate可)                   |
| **Full Route Cache**    | HTMLやRSC payload | Server | レンダリングコストの最適化                 | 永続的 (revalidate可)                   |
| **Router Cache**        | RSC Payload       | Client | ナビゲーションごとのリクエスト削減         | ユーザーセッション・時間 (revalidate可) |

![Next.jsのCacheの多層化](/images/nextjs-caching-history/cache-layer.png)

App Router初期のCacheは様々な要因が重なり、開発者に多くの混乱をもたらしました。

### 混乱ポイント: `fetch`とCache

## 構成

- v13: App Router初期のCache
- v14, v15: Cacheの改善
- v16: Cache Components
- 考察: "use cache"に学ぶ抽象化とOSSのプロセス
- v17~:
