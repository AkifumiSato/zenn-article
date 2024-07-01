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

### 低速な通信

通信時間はリクエスト元からリクエスト先までの物理的距離とネットワーク速度に強く依存します。クライアント・サーバー間の通信の場合物理的距離は遠く、クライアントの通信環境は不安定だったり低速なことも多いため、結果クライアント・サーバー間の通信は低速な傾向にあります。

### 実装コスト

クライアントサイドのデータフェッチでは[Reactが公式に推奨](https://ja.react.dev/reference/react/useEffect#what-are-good-alternatives-to-data-fetching-in-effects)してるように、多くの場合キャッシュ機能を搭載した3rd partyライブラリを利用します。一方リクエスト先に当たるAPIは、パブリックなネットワークに公開するためより堅牢なセキュリティが求められます。

これらの理由からクライアントサイドのデータフェッチには、ライブラリの学習・責務設計・セキュリティチェックの実装など様々なコストが発生します。

### バンドルサイズ

クライアントサイドでデータフェッチを行うために、データフェッチライブラリ・データフェッチの実装・バリデーションなど多岐にわたるコードがクライアントへ送信されるバンドルに含まれます。また、エラー時UIなどの通信の結果次第では利用されないコードもバンドルに含まれます。

## 設計・プラクティス

[Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)は前述の問題を解消するべくReactチームが取り組んだ結果生まれた概念であり、App Routerにおいて**データフェッチはServer Components上で行うことがベストプラクティス**とされています。

https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#fetching-data-on-the-server

これにより、以下のようなメリットを得られます。

### 高速なバックエンドアクセス

自身の管理下にあるBFFとAPIサーバー間の通信は多くの場合、同一ネットワーク内にあるため高速で安定しています。外部にあるAPIサーバーとの通信においても、サーバー間通信は日本の場合多くが東京都内での通信になるため、比較的高速で安定してることが多いと考えられます。

### シンプルでセキュアな実装

Server Componentsは非同期関数をサポートしており、3rd partyライブラリなしでデータフェッチをシンプルに実装できます。

```tsx
export async function UserName({ id }) {
  const res = await fetch(`https://api.example.com/profiles/${id}`);
  const profile = await res.json();

  return <div>{profile.name}</div>;
}
```

また、データフェッチはサーバー側でのみ実行されるためAPIのパブリックなネットワーク公開は必須ではありません。

### バンドルサイズの軽減

Server Componentsの実行結果はhtmlやRSC Payloadとしてクライアントへ送信されます。そのため、前述のような

> データフェッチライブラリ、データフェッチの実装、バリデーションなど多岐にわたるコード
> ...
> エラー時UIなどの通信の結果次第では利用されないコード

は一切バンドルには含まれません。

## トレードオフ

ユーザー操作に基づくデータフェッチはServer Componentsで行うことが困難な場合があります。これについては後述の[ユーザー操作とデータフェッチ](part_1_interactive_fetch)を参照してください。
