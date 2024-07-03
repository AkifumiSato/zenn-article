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

### デメリット1 パフォーマンスと設計のトレードオフ

クライアント・サーバー間の通信は物理的距離や不安定なネットワーク環境の影響で低速になりがちです。そのためパフォーマンス観点では通信回数が少ないことが望ましいですが、通信回数を減らすことは設計観点とトレードオフになりがちです。

Restful APIにおいて通信回数を優先するとGod APIと呼ばれる責務が**大きなAPI**になりがちで、変更容易性やAPIのパフォーマンス問題が起きやすい傾向にあります。一方**小さなAPI**では、データフェッチをコロケーション(コードをできるだけ関連性のある場所に配置すること)してコンポーネントが必要とする情報をカプセル化することなどのメリットを得られますが、通信回数が増えたりデータフェッチのウォーターフォールが発生しやすいなど、クライアントサイドでのパフォーマンス劣化の要因になりえます。

### デメリット2 様々な実装コスト

クライアントサイドのデータフェッチでは[Reactが公式に推奨](https://ja.react.dev/reference/react/useEffect#what-are-good-alternatives-to-data-fetching-in-effects)してるように、多くの場合キャッシュ機能を搭載した3rd partyライブラリを利用します。一方リクエスト先に当たるAPIは、パブリックなネットワークに公開するためより堅牢なセキュリティが求められます。

これらの理由からクライアントサイドのデータフェッチには、ライブラリの学習・責務設計・セキュリティチェックの実装など様々なコストが発生します。

### デメリット3 バンドルサイズの増加

クライアントサイドでデータフェッチを行うために、データフェッチライブラリ・データフェッチの実装・バリデーションなど多岐にわたるコードがクライアントへ送信されるバンドルに含まれます。また、エラー時UIなどの通信の結果次第では利用されないコードもバンドルに含まれます。

### クライアントサイドデータフェッチのベストプラクティス

上述の「デメリット1 パフォーマンスと設計のトレードオフ」を解消する手段として、GraphQLならコロケーションしつつ、[Render-as-you-fetch](https://17.reactjs.org/docs/concurrent-mode-suspense.html#approach-3-render-as-you-fetch-using-suspense)すれば**パフォーマンスと小さなAPI設計相当を実現**できます。こちらについては、以下の記事が参考になります。

https://quramy.medium.com/render-as-you-fetch-incremental-graphql-fragments-70e643edd61e

ただしこれをNext.jsなどに組み合わせる場合、クライアントルーターと連携させる必要があります。また、GraphQLライブラリのバンドルサイズも当然含まれることになるので、この場合も「デメリット2: 様々な実装コスト」「デメリット3: バンドルサイズの増加」は解消されません。

## 設計・プラクティス

Reactチームは前述の問題を個別の問題と捉えず、根本的には「Reactがサーバーをうまく活用できてないこと」が問題であると捉えて解決を目指しました。その結果生まれたのが[Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)であり、Server ComponentsがデフォルトのApp Routerにおいては**データフェッチはServer Components上で行うことがベストプラクティス**とされています。

https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#fetching-data-on-the-server

これにより、以下のようなメリットを得られます。

### 高速なバックエンドアクセス

自身の管理下にあるBFFとAPIサーバー間の通信は多くの場合、同一ネットワーク内にあるため高速で安定しています。外部にあるAPIサーバーとの通信においても、サーバー間通信は日本の場合多くが東京都内での通信になるため、比較的高速で安定してることが多いと考えられます。

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

ユーザー操作に基づくデータフェッチはServer Componentsで行うことが困難な場合があります。これについては後述の[ユーザー操作とデータフェッチ](part_1_interactive_fetch)を参照してください。
