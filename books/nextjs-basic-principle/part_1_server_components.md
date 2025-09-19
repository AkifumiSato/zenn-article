---
title: "データフェッチ on Server Components"
---

## 要約

データフェッチはClient Componentsではなく、Server Componentsで行いましょう。

## 背景

Reactにおけるコンポーネントは従来クライアントサイドでの処理を主体としていたため、クライアントサイドにおけるデータフェッチのためのライブラリや実装パターンが多く存在します。

- [SWR↗︎](https://swr.vercel.app/)
- [React Query↗︎](https://react-query.tanstack.com/)
- GraphQL
  - [Apollo Client↗︎](https://www.apollographql.com/docs/react/)
  - [Relay↗︎](https://relay.dev/)
- [tRPC↗︎](https://trpc.io/)
- etc...

しかしクライアントサイドでデータフェッチを行うことは、多くの点でデメリットを伴います。

### パフォーマンスと設計のトレードオフ

クライアント・サーバー間の通信は、物理的距離や不安定なネットワーク環境の影響で多くの場合低速です。そのため、パフォーマンス観点では通信回数が少ないことが望ましいですが、通信回数とシンプルな設計はトレードオフになりがちです。

REST APIにおいて通信回数を優先すると**God API**と呼ばれる責務が大きなAPIになりがちで、変更容易性やAPI自体のパフォーマンス問題が起きやすい傾向にあります。一方責務が小さい細粒度なAPIは**Chatty API**(おしゃべりなAPI)と呼ばれ、データフェッチをコロケーション^[コードをできるだけ関連性のある場所に配置することを指します。]してカプセル化などのメリットを得られる一方、通信回数が増えたりデータフェッチのウォーターフォールが発生しやすく、Webアプリのパフォーマンス劣化要因になりえます。

### 様々な実装コスト

クライアントサイドのデータフェッチでは多くの場合、[Reactが推奨↗︎](https://ja.react.dev/reference/react/useEffect#what-are-good-alternatives-to-data-fetching-in-effects)してるようにキャッシュ機能を搭載した3rd partyライブラリを利用します。一方リクエスト先に当たるAPIは、パブリックなネットワークに公開するためより堅牢なセキュリティが求められます。

これらの理由からクライアントサイドでデータフェッチする場合には、3rd partyライブラリの学習・責務設計・API側のセキュリティ対策など様々な開発コストが発生します。

### バンドルサイズの増加

クライアントサイドでデータフェッチを行うには、3rd partyライブラリ・データフェッチの実装・バリデーションなど、多岐にわたるコードがバンドルされクライアントへ送信されます。また、通信結果次第では利用されないエラー時のUIなどのコードもバンドルに含まれがちです。

## 設計・プラクティス

Reactチームは前述の問題を個別の問題と捉えず、根本的にはReactがサーバーをうまく活用できてないことが問題であると捉えて解決を目指しました。その結果生まれたのが**React Server Components**（**RSC**）アーキテクチャです。

https://ja.react.dev/reference/rsc/server-components

Next.jsはRSCをサポートしており、[データフェッチはServer Componentsで行う↗︎](https://nextjs.org/docs/app/getting-started/fetching-data)ことがベストプラクティスとされています。

:::message alert
「Server Componentsには`"use server"`が必要」という誤解が散見されますが、これは**誤り**です。`"use server"`は関数を[Server Functions↗︎](https://ja.react.dev/reference/rsc/server-functions)としてマークし、クライアントサイドから呼び出し可能にするものであり、Server Componentsに指定するためのものではありません。

詳しくは[クライアントとサーバーのバンドル境界](part_2_bundle_boundary)を参照ください。
:::

データフェッチをServer Componentsで行うにより、以下のようなメリットを得られます。

### 高速なバックエンドアクセス

Next.jsサーバーとAPIサーバー間の通信は、多くの場合高速で安定しています。特に、APIが同一ネットワーク内や同一データセンターに存在する場合は非常に高速です。APIサーバーが外部にある場合も、多くの場合は首都圏内で高速なネットワーク回線を通じての通信になるため、比較的高速で安定してることが多いと考えられます。

### シンプルでセキュアな実装

Server Componentsは非同期関数をサポートしており、3rd partyライブラリなしでデータフェッチをシンプルに実装できます。

```tsx
export async function ProductTitle({ id }) {
  const res = await fetch(`https://dummyjson.com/products/${id}`);
  const product = await res.json();

  return <div>{product.title}</div>;
}
```

これはServer Componentsがサーバー側で1度だけレンダリングされ、従来のようにクライアントサイドで何度もレンダリングされることを想定しなくて良いからこそできる設計です。

また、データフェッチはサーバー側でのみ実行されるため、APIをパブリックなネットワークで公開することは必須ではありません。プライベートなネットワーク内でのみバックエンドAPIへアクセスするようにすれば、セキュリティリスクや対策コストを軽減できます。

### バンドルサイズの軽減

Server Componentsの実行結果はHTMLやRSC Payloadとしてクライアントへ送信されます。そのため、前述のような

> 3rd partyライブラリ・データフェッチの実装・バリデーションなど、多岐にわたるコード
> ...
> エラー時のUIなどのコード

は一切バンドルには含まれません。

## トレードオフ

### ユーザー操作とデータフェッチ

ユーザー操作に基づくデータフェッチはServer Componentsで行うことが困難な場合があります。詳細は後述の[ユーザー操作とデータフェッチ](part_1_interactive_fetch)を参照してください。

### GraphQLとの相性の悪さ

RSCにGraphQLを組み合わせることは**メリットよりデメリットの方が多くなる**可能性があります。

GraphQLはその特性上、前述のようなパフォーマンスと設計のトレードオフが発生しませんが、RSCも同様にこの問題を解消するため、これをメリットとして享受できません。それどころか、RSCとGraphQLを協調させるための知見やライブラリが一般に不足してるため、実装コストが高くバンドルサイズも増加するなど、デメリットが多々含まれます。

:::message
RSCの最初のRFCは、Relayの初期開発者の1人でGraphQLを通じてReactにおけるデータフェッチのベストプラクティスを追求してきた[Joe Savona氏↗︎](https://twitter.com/en_js)によって提案されました。そのため、RSCはGraphQLの持っているメリットや課題を踏まえて設計されているという**GraphQLの精神的後継**の側面を持ち合わせていると考えることができます。

より詳しくは、[Reactチームが見てる世界、Reactユーザーが見てる世界↗︎](https://zenn.dev/akfm/articles/react-team-vision)で解説しているので、ご参照ください。
:::
