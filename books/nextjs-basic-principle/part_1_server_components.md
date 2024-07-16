---
title: "データフェッチ on Server Components"
---

## 要約

データフェッチは基本的にServer Componentsで行いましょう。

## 背景

Reactは従来クライアントサイドでの処理を主体としていたため、クライアントサイドにおけるデータ取得のためのライブラリや実装パターンが多く存在します。

- [SWR](https://swr.vercel.app/)
- [React Query](https://react-query.tanstack.com/)
- GraphQL
  - [Apollo Client](https://www.apollographql.com/docs/react/)
  - [Relay](https://relay.dev/)
- [tRPC](https://trpc.io/)
- etc...

しかしクライアントサイドでデータ取得を行うことは、多くの点でデメリットを伴います。

### パフォーマンスと設計のトレードオフ

クライアント・サーバー間の通信は物理的距離や不安定なネットワーク環境の影響で低速になりがちです。そのためパフォーマンス観点では通信回数が少ないことが望ましいですが、通信回数を減らすことは設計観点とトレードオフになりがちです。

REST APIにおいて通信回数を優先するとGod APIと呼ばれる責務が**大きなAPI**になりがちで、変更容易性やAPIのパフォーマンス問題が起きやすい傾向にあります。一方**小さなAPI**では、データフェッチをコロケーション(コードをできるだけ関連性のある場所に配置すること)してコンポーネントが必要とする情報をカプセル化することなどのメリットを得られますが、通信回数が増えたりデータフェッチのウォーターフォールが発生しやすいなど、クライアントサイドでのパフォーマンス劣化の要因になりえます。

### 様々な実装コスト

クライアントサイドのデータフェッチでは[Reactが公式に推奨](https://ja.react.dev/reference/react/useEffect#what-are-good-alternatives-to-data-fetching-in-effects)してるように、多くの場合キャッシュ機能を搭載した3rd partyライブラリを利用します。一方リクエスト先に当たるAPIは、パブリックなネットワークに公開するためより堅牢なセキュリティが求められます。

これらの理由からクライアントサイドのデータフェッチには、ライブラリの学習・責務設計・セキュリティチェックの実装など様々なコストが発生します。

### バンドルサイズの増加

クライアントサイドでデータフェッチを行うために、データフェッチライブラリ・データフェッチの実装・バリデーションなど多岐にわたるコードがクライアントへ送信されるバンドルに含まれます。また、エラー時UIなどの通信の結果次第では利用されないコードもバンドルに含まれます。

## 設計・プラクティス

Reactチームは前述の問題を個別の問題と捉えず、根本的には「Reactがサーバーをうまく活用できてないこと」が問題であると捉えて解決を目指しました。その結果生まれたのが[Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)であり、Server ComponentsがデフォルトのApp Routerにおいては**データフェッチはServer Components上で行うことがベストプラクティス**とされています。

https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#fetching-data-on-the-server

これにより、以下のようなメリットを得られます。

### 高速なバックエンドアクセス

Next.jsサーバーとAPIサーバー間の通信は多くの場合高速で安定しています。APIが同一ネットワーク内や同一データセンターに存在する場合は非常に高速で、外部にあるAPIサーバーとの通信においても多くの場合首都圏内で高速なネットワーク回線を通じての通信になるため、比較的高速で安定してることが多いと考えられます。

### シンプルでセキュアな実装

Server Componentsは非同期関数をサポートしており、3rd partyライブラリなしでデータフェッチをシンプルに実装できます。

```tsx
export async function ProductTitle({ id }) {
  const res = await fetch(`https://dummyjson.com/products/${id}`);
  const product = await res.json();

  return <div>{product.title}</div>;
}
```

これはServer Componentsがサーバー側でリクエスト時のみレンダリングされ、従来のようにクライアントサイドで何度もレンダリングされることを想定しなくて良いからこそできる設計です。また、データフェッチはサーバー側でのみ実行されるためAPIのパブリックなネットワーク公開は必須ではありません。

### バンドルサイズの軽減

Server Componentsの実行結果はhtmlやRSC Payloadとしてクライアントへ送信されます。そのため、前述のような

> データフェッチライブラリ、データフェッチの実装、バリデーションなど多岐にわたるコード
> ...
> エラー時UIなどの通信の結果次第では利用されないコード

は一切バンドルには含まれません。

## トレードオフ

### ユーザー操作とデータフェッチ

ユーザー操作に基づくデータフェッチはServer Componentsで行うことが困難な場合があります。これについては後述の[ユーザー操作とデータフェッチ](part_1_interactive_fetch)を参照してください。

### GraphQLとの相性の悪さ

React Server Components(RSC)にGraphQLを組み合わせることは**メリットよりデメリットの方が多くなる**可能性があります。

GraphQLは大きなAPIと小さなAPIのトレードオフがないので[パフォーマンスと設計のトレードオフ](#パフォーマンスと設計のトレードオフ)が発生しませんが、RSCも同様にこの問題を解消するため、これをメリットとして享受できません。一方RSCとGraphQLの組み合わせは協調させるための知見やライブラリが一般に不足してるため実装コストが高く、GraphQLライブラリなどを含むことでバンドルサイズも増加するなど、デメリットが含まれます。

:::message
RSCの最初のRFCは、Relayの初期開発者の1人でGraphQLを通じてReactにおけるデータフェッチのベストプラクティスを追求してきた[Joe Savona氏](https://twitter.com/en_js)によって提案されました。そのため、RSCはGraphQLの持っているメリットや課題を踏まえて設計されているという**GraphQLの精神的後継**の側面を持ち合わせていると考えることができます。
:::
