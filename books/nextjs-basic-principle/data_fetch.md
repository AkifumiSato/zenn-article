---
title: "データ取得はServer Componentsで、必要な場所で"
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

しかしクライアントサイドでデータ取得を行うことは、多くの点でデメリットを伴います。クライアントサイドでのデータ取得はサーバー側と比べると開始タイミングが遅くなってしまい、またデータソースとの物理的な距離も遠くネットワークの安定性も低いため、パフォーマンス的に不利です。セキュリティ的にもクライアントサイドから参照できるAPIを公開する必要があります。

Pages Routerではこれらの課題を、[getServerSideProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)や[getStaticProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-static-props)といったサーバー側でのデータ取得アプローチを採用することで解消しました。しかし、ページのレンダリング開始前にすべてのデータ取得を終える必要のあるこの設計は、**データ取得後のProps バケツリレー**(Props Drilling)を生み出しました。

以下の実装例では`<Page>`から`<ComponentA>`、`<ComponentB>`、`<ComponentC>`と`data`をそのまま渡している様子が見て撮れます。これがいわゆるバケツリレーです。

```tsx
export async function getServerSideProps() {
  const data = await fetch("https://dummyjson.com/data").then((res) =>
    res.json(),
  );

  return { props: { data } };
}

export default function Page({ data }) {
  return (
    <>
      <ComponentA data={data} />
      {/* ... */}
    </>
  );
}

function ComponentA({ data }) {
  return (
    <>
      <ComponentB data={data} />
      {/* ... */}
    </>
  );
}

function ComponentB({ data }) {
  return (
    <>
      <ComponentC data={data} />
      {/* ... */}
    </>
  );
}

function ComponentC({ data }) {
  return <p>data id: {data.id}</p>;
}
```

## ベストプラクティス

App Routerは[Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)をサポート・基本としています。Server Componentsはサーバー側でのみレンダリングされるため、より積極的なサーバー活用を可能にします。RFCの[Motivation](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md#motivation)によると、バンドルサイズの低減、バックエンドへのフルアクセス、コード分割の自動化など複数の課題に対し、統一されたソリューションとして採用されたのが**React Server Components**アーキテクチャです。

Server Componentsは非同期関数をサポートしており、`fetch`を直接扱うことが可能です。前述の例で考えてみると、`<Page>`/`<ComponentA>`/`<ComponentB>`では`data`はバケツリレーするのみだったので、`<ComponentC>`を以下のように書き換えることができます。

```tsx
async function ComponentC() {
  const data = await fetch("https://dummyjson.com/data").then((res) =>
    res.json(),
  );

  return <p>data id: {data.id}</p>;
}
```

これにより従来ページの外側でしかできなかったデータ取得が、**データを参照したいコンポーネント、もしくはその近く**で行えるようになりました。従来のReactコンポーネントはhtml/css/jsをカプセル化していましたが、React Server Componentsにおいてはデータ取得まで含めてカプセル化できることになります。

### Request Memoization

しかし、末端のコンポーネントでデータ取得を行うと重複するリクエストが多発するのではないかと懸念される方もいらっしゃると思います。App Routerではこのような重複リクエストを避けるため、[Request Memoization](https://nextjs.org/docs/app/building-your-application/caching#request-memoization)が実装されています。

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

Request Memoizationはメモ化なので、当然ながら`fetch()`の引数に同じURLとオプション指定が必要です。上記例における`getItem()`のように、複数のコンポーネントで実行しうるデータ取得は関数などに抽出しておくことが望ましいでしょう。

## 結論

このように、App RouterにおけるServer Componentsでのデータ取得は非常に優れた設計とメリットを併せ持っています。そのため、App Routerでは可能な限り**データ取得はServer Componentsで行うことを推奨**しています。これはNext.jsの[Patterns and Best Practices](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#fetching-data-on-the-server)でも示されています。

データ取得は、実際にデータを参照するコンポーネント自体・もしくは近くのServer Componentsで行うようにしましょう。

## 例外

### サイト内で利用する外部SaaSのSDK都合

サイト内で利用する外部SaaSのSDK都合で、クライアントサイドでデータ取得や操作を行わないことはあるかもしれません。しかし可能なら、Server Componentsで処理する方が良いでしょう。

### `useActionState`とServer Actionsを組み合わせて使う場合

ユーザーの入力に基づいてサーバー側でデータ取得を行いたいケースにおいては、URLを変更してServer Componentsを再実行する手段もありますが、[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)と[`useActionState`](https://react.dev/reference/react/useActionState)(旧: `useFormState`)を利用して実装することも可能です。

以下はユーザーの入力に基づいて商品を検索する実装例です。

```ts
// app/actions.ts
"use server";

export async function searchProducts(
  _prevState: Product[],
  formData: FormData,
) {
  const name = formData.get("name") as string;
  const res = await fetch(`https://dummyjson.com/products/search?q=${name}`);
  const { products } = (await res.json()) as { products: Product[] };

  return products;
}

type Product = {
  id: number;
  title: string;
};
```

```tsx
// app/form.tsx
"use client";

import { useActionState } from "react-dom";
import { searchProducts } from "./actions";

export default function Form() {
  const [products, action] = useActionState(searchProducts, []);

  return (
    <>
      <form action={action}>
        <label htmlFor="name">
          Search Product:&nbsp;
          <input type="text" id="name" name="name" />
        </label>
        <button type="submit">Submit</button>
      </form>
      <ul>
        {products.map((product) => (
          <li key={product.id}>{product.title}</li>
        ))}
      </ul>
    </>
  );
}
```
