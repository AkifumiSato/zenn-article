---
title: "データ取得は近くのServer Componentsで行う"
---

## 背景

Reactは従来クライアントサイドでの処理を主体としていました。そのため、クライアントサイドにおけるデータ取得のためのライブラリや実装パターンが多く存在します。

- [SWR](https://swr.vercel.app/)
- [React Query](https://react-query.tanstack.com/)
- GraphQL
  - [Apollo Client](https://www.apollographql.com/docs/react/)
  - [Relay](https://relay.dev/)
- [tRPC](https://trpc.io/)
- etc...

しかしクライアントサイドでデータ取得を行うことは、多くの点でデメリットを伴います。クライアントサイドでのデータ取得はサーバー側と比べると開始タイミングが遅くなってしまい、またデータソースとの物理的な距離も遠くネットワークの安定性も低いため、パフォーマンス的に不利です。セキュリティ的にもクライアントサイドから参照できるAPIを公開する必要があり、SEO的にもクライアントサイド処理に依存するコンテンツは不利になり得ます。

Pages Routerではこれらの課題を、[getServerSideProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)や[getStaticProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-static-props)といったサーバー側でのデータ取得アプローチとSSR/SSG/ISRなどのレンダリングモデルを採用することで解消しました。しかし、ページのレンダリング開始前にすべてのデータ取得を終える必要のあるこれらの設計は、**データ取得後のProps バケツリレー**(Props Drilling)を生み出しました。

```tsx
export default function Page({ data }) {
  return <ComponentA data={data} />;
}

export async function getServerSideProps() {
  const data = await fetch("https://dummyjson.com/data").then((res) =>
    res.json(),
  );

  return { props: { data } };
}

function ComponentA({ data }) {
  return (
    <>
      <h1>ComponentA</h1>
      <ComponentB data={data} />
    </>
  );
}

function ComponentB({ data }) {
  return (
    <>
      <h2>ComponentB</h2>
      <ComponentC data={data} />
    </>
  );
}

// ...
```

## プラクティス

App Routerは[Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)をサポート・基本としています。Server Componentsはサーバー側でのみレンダリングされるため、より積極的なサーバー活用を可能にします。RFCの[Motivation](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md#motivation)によると、バンドルサイズの低減、バックエンドへのフルアクセス、コード分割の自動化など複数の課題に対し、統一されたソリューションとして採用されたのがReact Server Componentsアーキテクチャです。

Server Componentsは非同期関数をサポートしており、`fetch`を直接扱うことが可能です。

```tsx
export async function LeafComponent() {
  const product = await fetch("https://dummyjson.com/products/1").then((res) =>
    res.json(),
  );

  return <>...</>;
}
```

これにより従来ページの外側でしかできなかったデータ取得が、**データを参照したいコンポーネントの近くで行うことができる**ようになりました。従来のReactコンポーネントはhtml/css/jsをカプセル化していましたが、React Server Componentsにおいてはデータ取得まで含めてカプセル化できることになります。

しかし、末端のコンポーネントでデータ取得を実装すると重複するリクエストが多発するのではないかと懸念される方もいらっしゃると思います。App Routerではこのような重複リクエストを避けるため、[Request Memoization](https://nextjs.org/docs/app/building-your-application/caching#request-memoization)が実装されています。

```ts
async function getItem() {
  const res = await fetch("https://.../item/1");
  return res.json();
}

// <Component1 />
// <Component2 />
// の順で呼び出された場合

async function Component1() {
  const item = await getItem(); // cache MISS
  // ...
}

async function Component2() {
  const item = await getItem(); // cache HIT
  // ...
}
```

Request Memoizationはメモ化なので、当然ながら`fetch()`の引数に同じURLとオプション指定が必要です。複数のコンポーネントで実行しうるデータ取得は関数として抽出(上記例における`getItem()`)しておくことが望ましいでしょう。

このように、App RouterにおけるServer Componentsでのデータ取得は非常に優れた設計とメリットを併せ持っています。そのため、App Routerでは可能な限り**データ取得はServer Componentsで行うことを推奨**しています。これはNext.jsの[Patterns and Best Practices](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#fetching-data-on-the-server)でも示されています。

データ取得は、実際にデータを参照するコンポーネント自体・もしくは近くのServer Componentsで行うようにしましょう。

## 例外

### サイト内で利用する外部SaaSのSDK都合

サイト内で利用する外部SaaSのSDK都合で、クライアントサイドでデータ取得や操作を行わないことはあるかもしれません。しかし可能なら、Server Componentsで処理する方が良いでしょう。
