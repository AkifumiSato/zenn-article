---
title: "細粒度のREST API設計"
---

## 要約

バックエンドのAPIは細粒度リソース単位のREST APIに設計しましょう。

:::message
このchapterの主題は「App Routerが呼び出すバックエンドAPIの設計」の話です。「App Routerの[Route Handler](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)として実装するAPIの設計」ではないのでご注意ください。
:::

## 背景

昨今のバックエンドAPI開発において、最もよく用いられる設計は[REST API](https://learn.microsoft.com/ja-jp/azure/architecture/best-practices/api-design)です。しかし、REST APIの設計においてリソース単位をどう設計するかは経験とスキルを必要とする、難しい問題です。

### リソース単位の粒度とトレードオフ

REST APIにおいてはリソース単位は**細粒度**、つまり細かくすることが基本ですが、細かくしすぎるといわゆるChatty API(おしゃべりなAPI)になり、アンチパターンとされています。`/users/[id]/username`のように、User情報の細かい単位でAPIエンドポイントが設定されてたら利用者側の実装が必要以上に冗長になってしまうことが想像できます。

一方でリソース単位を非正規化し**粗粒度**、つまり大きな単位にすると通信回数を減らすことができますが、API単位が大きいほど汎用性・パフォーマンスの劣化など様々なトレードオフが発生します。これはいわゆるGod API(神API)と呼ばれるアンチパターンです。`/users/1`で常にUserが執筆したブログ記事一覧やそのコメントまで取れてしまったら、明らかに情報を取得しすぎていると感じられることでしょう。

粒度の適度なバランスをとることが重要で、そこに経験とスキルが求められます。

### 粒度のバランスと利用者都合

バックエンドAPIの粒度を設計する時、利用者側であるフロントエンド都合を考慮して決定することが多々あります。

フロントエンド側の実装がクライアントサイドからデータフェッチを行うような設計の場合、粗粒度なAPI設計の方がパフォーマンス上有利となります。クライアント・サーバー間の通信は物理的距離や不安定なネットワーク環境の影響で低速になりがちなために、通信回数は少ない方が好ましいためです。

一方Pages Routerの[getServerSideProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)などを利用する場合はサーバー間通信のため通信回数はある程度無視できますが、実装容易性の観点から粗粒度単位で設計される傾向があります。Pages Routerの設計上ページの外で一度にデータフェッチを行う必要があるため、データフェッチと参照が遠く、細粒度のAPIだと実装が冗長・煩雑になってしまうからです。

## 設計・プラクティス

App RouterにおいてはServer Componentsによってデータフェッチのコロケーションや分割が容易に可能になったため、データ取得とデータ参照のコードが近くなり、コードやロジックの重複が発生しづらくなりました。このため、App Routerは**細粒度で設計されたREST APIと非常に相性が良い**と言えます。

バックエンドAPIの設計観点から言っても細粒度で設計されたREST APIの実装はシンプルに実装できることが多く、メリットとなるはずです。

## トレードオフ

### バックエンドとの通信回数

前述の通り、サーバー間通信は多くの場合高速で安定しています。そのため通信回数が多いことはデメリットになりづらいと言えますが、アプリケーション特性にもよるので実際には注意が必要です。

[並行データフェッチ](part_1_concurrent_fetch)や[N+1とDataLoader](part_1_data_loader)で述べたプラクティスや、データフェッチ単位のキャッシュである[Data Cache](https://nextjs.org/docs/app/building-your-application/caching)を活用して、通信頻度やパフォーマンスを最適化しましょう。

### バックエンドAPI開発チームの理解

バックエンドAPIには複数の利用者がいる場合もあるため、Next.jsが細粒度のAPIの方が都合がいいからといって一存で決めれるとは限りません。バックエンド開発チームの経験則や価値観もあります。

しかし、細粒度のAPIにすることはフロントエンド開発チームにとってもバックエンド開発チームにとってもメリットが大きく、無碍にできない要素なはずです。最終的な判断がバックエンド開発チームにあるとしても、しっかりメリットやNext.js側の背景を伝え理解を得るべく努力しましょう。