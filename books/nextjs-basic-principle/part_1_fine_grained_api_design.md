---
title: "細粒度のREST API設計"
---

## 要約

バックエンドのREST API設計は、Next.js側の設計にも大きく影響をもたらします。App Router(React Server Components)におけるバックエンドAPIは、細粒度な単位に分割されていることが望ましく、可能な限りこれを意識した設計を行いましょう。

:::message alert
このchapterの主題は「App Routerが呼び出すバックエンドAPIの設計」です。「App Routerの[Route Handler](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)として実装するAPIの設計」ではないのでご注意ください。
:::

## 背景

昨今のバックエンドAPI開発において、最もよく用いられる設計は[REST API](https://learn.microsoft.com/ja-jp/azure/architecture/best-practices/api-design)です。REST APIのリソース単位は、識別子を持つデータやオブジェクト単位でできるだけ細かく**細粒度**に分割することが基本です。しかし、細粒度のAPIは利用者システムとの通信回数が多くなってしまうため、これを避けるためにより大きな**粗粒度**単位でAPI設計することがよくあります。

### リソース単位の粒度とトレードオフ

細粒度なAPIで通信回数が多くなることはChatty API（おしゃべりなAPI）と呼ばれ、逆に粗粒度なAPIで汎用性に乏しい状態はGod API（神API）と呼ばれます。これらはそれぞれアンチパターンとされることがありますが、実際には観点次第で最適解が異なるので、一概にアンチパターンなのではなくそれぞれトレードオフが伴うと捉えるべきです。

| リソース単位の粒度 | 設計観点 | パフォーマンス(低速な通信) | パフォーマンス(高速な通信) |
| ------------------ | -------- | -------------------------- | -------------------------- |
| 細粒度             | ✅       | ❌                         | ✅                         |
| 粗粒度             | ❌       | ✅                         | ✅                         |

### ページと密結合になりがちな粗粒度単位

App Router登場以前のNext.jsやReactアプリケーションに対するバックエンドAPIでは、API設計がページと密結合になり、汎用性や保守性に乏しくなるケースが多々見られました。

クライアントサイドでデータフェッチを行う場合、クライアント・サーバー間の通信は物理的距離や不安定なネットワーク環境の影響で低速になりがちなため、通信回数はパフォーマンスに大きく影響します。そのため、ページごとに1度の通信で完結するべく粗粒度なAPI設計が採用されることがありました。

一方、Pages Routerの[`getServerSideProps()`](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)を利用したBFFとAPIのサーバー間においては、高速なネットワークを介した通信となるため、通信回数がパフォーマンスボトルネックになる可能性は低くなります。しかし、`getServerSideProps()`はページ単位で定義するため、APIにページ単位の要求が反映され、粗粒度なAPI設計になるようなケースがよくありました。

## 設計・プラクティス

App Routerにおいては、Server Componentsによってデータフェッチのコロケーションや分割が容易になったためコードやロジックの重複が発生しづらくなりました。このため、App Routerは細粒度で設計されたREST APIと**非常に相性が良い**と言えます。

バックエンドAPIの設計観点から言っても、細粒度で設計されたREST APIの実装はシンプルに実装できることが多く、メリットとなるはずです。

### React Server ComponentsとGraphQLのアナロジー

細粒度なAPI設計はReact Server Componentsで初めて注目されたものではなく、従来からGraphQL BFFに対するバックエンドAPIで好まれる傾向にありました。

https://x.com/koichik/status/1825711304834953337

このように、React Server ComponentsとGraphQLには共通の思想が見えます。

[_データフェッチ on Server Components_](part_1_server_components)でも述べたように、React Server Componentsの最初のRFCはRelayの初期開発者の1人でGraphQLを通じてReactにおけるデータフェッチのベストプラクティスを追求してきた[Joe Savona氏](https://twitter.com/en_js)が提案しており、React Server Componentsには**GraphQLの精神的後継**という側面があります。

このようなReact Server ComponentsとGraphQLにおける類似点については、以下のQuramyさんの記事で詳しく解説されているので、興味がある方は参照してみてください。

https://quramy.medium.com/react-server-components-%E3%81%A8-graphql-%E3%81%AE%E3%82%A2%E3%83%8A%E3%83%AD%E3%82%B8%E3%83%BC-89b3f5f41a01

## トレードオフ

### バックエンドとの通信回数

前述の通り、サーバー間通信は多くの場合高速で安定しています。そのため、通信回数が多いことはデメリットになりづらいと言えますが、アプリケーション特性にもよるので実際には注意が必要です。

[_並行データフェッチ_](part_1_concurrent_fetch)や[_N+1とDataLoader_](part_1_data_loader)で述べたプラクティスや、データフェッチ単位のキャッシュである[Data Cache](https://nextjs.org/docs/app/building-your-application/caching)を活用して、通信頻度やパフォーマンスを最適化しましょう。

### バックエンドAPI開発チームの理解

バックエンドAPIには複数の利用者がいる場合もあるため、Next.jsが細粒度のAPIの方が都合がいいからといって一存で決めれるとは限りません。バックエンド開発チームの経験則や価値観もあります。

しかし、細粒度のAPIにすることはフロントエンド開発チームにとってもバックエンド開発チームにとってもメリットが大きく、無碍にできない要素なはずです。最終的な判断がバックエンド開発チームにあるとしても、しっかりメリットやNext.js側の背景を伝え理解を得るべく努力しましょう。
