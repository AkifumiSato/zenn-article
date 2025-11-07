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

Next.jsは従来より、デフォルトで高いパフォーマンスを実現するフレームワークであることを重視してきました。App Routerにおいても、**積極的なCache活用**をはじめさまざまなチューニングにより、高いパフォーマンスの実現を目指してきました。一方で、積極的なCache活用や複雑すぎる影響範囲は、開発者に多くの混乱をもたらしました。

v16で導入された[Cache Components](https://nextjs.org/docs/app/getting-started/cache-components)は、Next.jsのCacheのメンタルモデルを大きく変更するものであり、これまでの多くの課題を解決しうる根本的な**リアーキテクチャ**です。この記事では、Cache Componentsが何を解決しようとしているのか、なぜリアーキテクチャに至ったのかなどの歴史的経緯を解説します。

## Next.jsの歴史

Next.jsには現在、Pages RouterとApp Router2つのRouterが同梱されています^[参考: [App Router and Pages Router](https://nextjs.org/docs#app-router-and-pages-router)]。2016年に発表された初期のNext.jsは現在Pages Routerと呼ばれるものであり、その後[React Server Components](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)をサポートした新たなRouterとして、App Routerが開発されました。

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

Next.jsは従来より、パフォーマンスを重視してきました。SSGが注目されてからは、Next.jsもSSGサポートやISRの発明など、静的化によるパフォーマンス最適化を積極的に行ってきました。

## Legacy Cache: App Router初期のCache

Pages Routerで培われた静的化によるパフォーマンス最適化思想はApp Routerでも引き継がれています。従来SSGやISRと呼ばれたページの静的化はCacheの1種として位置付けられ、より高度な最適化のために多層のCacheが導入されました。

以下は[公式の表↗︎](https://nextjs.org/docs/app/guides/caching#overview)を翻訳したものです。

| Mechanism               | What              | Where  | Purpose                                    | Duration                                |
| ----------------------- | ----------------- | ------ | ------------------------------------------ | --------------------------------------- |
| **Request Memoization** | APIレスポンスなど | Server | React Component treeにおけるデータの再利用 | リクエストごと                          |
| **Data Cache**          | APIレスポンスなど | Server | ユーザーやデプロイをまたぐデータの再利用   | 永続的 (revalidate可)                   |
| **Full Route Cache**    | HTMLやRSC payload | Server | レンダリングコストの最適化                 | 永続的 (revalidate可)                   |
| **Router Cache**        | RSC Payload       | Client | ナビゲーションごとのリクエスト削減         | ユーザーセッション・時間 (revalidate可) |

![Next.jsのCacheの多層化](/images/nextjs-caching-history/cache-layer.png)

これらはそれぞれ最適化の観点が異なるため、理解すれば高度なチューニングや設計が可能です。一方で多層のCacheはそれ自体が複雑なため、バグやドキュメント不足、高い設計難易度など開発者に多くの負担を強いることとなりました。

### 課題1: 予想困難な`fetch`

TBW

### 課題2: 複雑な設定、不足した設定

TBW

### 課題3: バグや脆弱性

TBW

## Cache Improvement: 議論と改善

## Cache Re-Architecture: 根本的な改善

## 考察

## 教訓と私見
