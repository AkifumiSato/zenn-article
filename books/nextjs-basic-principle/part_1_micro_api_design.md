---
title: "「小さなAPI」設計"
---

## 要約

バックエンドのAPIはユースケースに強く依存しない、リソース志向な「小さなAPI」を意識しましょう。

:::message
このchapterの主題は「App Routerが利用するバックエンドAPIの設計」の話です。App Routerにおける[Route Handler](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)の話題ではないのでご注意ください。
:::

## 背景

バックエンドAPI設計アプローチには大きく、ユースケース志向な**大きなAPI**とリソース志向で分割された**小さなAPI**の2つが考えられます。従来のPages Routerのバックエンドには、「大きなAPI」が採用される傾向にありました。

「大きなAPI」は、ページの改修にともなってAPIも変更しなければならなかったり、ページ間の共通化が密結合を生んでしまって不要な情報も大量にとってくるなど、様々な問題を引き起こします。

一方「小さなAPI」は変更容易性・可用性に強いですが、[getServerSideProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)などのページ単位の処理を通してデータ取得を行うような設計である以上、冗長な実装になりがちでした。

## 設計・プラクティス

App RouterにおいてはServer Componentsによってデータフェッチのコロケーションや分割が容易に可能になったため、データ取得とデータ参照のコードが近くなり、コードやロジックの重複が発生しづらくなりました。このため、App Routerは「小さなAPI」と非常に相性が良いと言えます。

一方バックエンドAPIの設計観点から言っても、リソース志向な「小さなAPI」はユースケース思考な「大きなAPI」よりも変更が少なくなり、シンプルに実装できるので大きなメリットを得られるはずです。

## トレードオフ

### バックエンドAPI開発チームの理解

バックエンドAPIの設計は、利用するフロントエンド開発チームが主導することもあればバックエンド開発チームが主導することもあります。この点は会社や組織によってまちまちでしょう。注意しなければならないのは、バックエンド開発チームが主導し、かつ「大きなAPI」設計になってしまった場合です。

前述の通り、「小さなAPI」にすることはフロントエンド開発チームにとってもバックエンド開発チームにとってもメリットが大きいです。しかし、これまでの開発経験が「大きなAPI」の方が慣れ親しんでるバックエンド開発チームは、自然と「大きなAPI」設計を採用してしまうこともあるでしょう。

この場合、フロントエンド開発チームがNext.js側の考慮や、「小さなAPI」のメリットをバックエンド開発チームに理解してもらうべく説明する必要があるでしょう。自ら設計を主導する立場にない場合、説明を理解してもらう難易度・コストは大きくなります。

しかし、フロントエンド側の都合を無考慮で「大きなAPI」設計になってしまうと、Next.js側の設計上大きな制約になってしまいます。双方に大きなメリットがあるということを、懇切丁寧に説明し、理解を得ましょう。