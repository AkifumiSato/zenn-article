---
title: "データ取得はServer Components、データ操作はServer Actions"
---

## 背景

従来ReactやNext.js Pages Routerにおけるデータ取得やデータ操作には、様々な実装パターンがありました。

- クライアントサイド
  - [SWR](https://swr.vercel.app/)
  - [React Query](https://react-query.tanstack.com/)
  - GraphQL
    - [Apollo Client](https://www.apollographql.com/docs/react/)
    - [Relay](https://relay.dev/)
  - [tRPC](https://trpc.io/)
  - etc...
- サーバーサイド
  - [getServerSideProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)
  - [getStaticProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-static-props)

クライアントサイドでデータ取得を行うことは、多くの点でデメリットを伴います。

- データ取得が始まるのがページロード後JavaScript実行時なので遅い
- データソース（データベースやAPIなど）とクライアントサイドでは物理的な距離が遠く、ネットワークも安定しないので遅くなりやすい
- クライアントサイドから参照できるAPIを公開する必要がある

Pages Routerではこれらの課題を`getServerSideProps`/`getStaticProps`によって解消してきました。しかし、ページのレンダリング前にデータ取得をすべて終える必要のあるこれらの設計は、以下のような課題を発生させました。

- データ取得後のPropsバケツリレー
- データ操作に関する公式アプローチの不在

## プラクティス

App Routerでは[Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)をサポート・基本としているため、コンポーネントを`async`にしつつ直接`await fetch(...)`のように実装することが可能です。これはサーバー側でのみ実行されるのでセキュアで高速、それでいてシンプルに実装することが可能です。

```tsx
export default async function Page() {
  const product = await fetch("https://dummyjson.com/products/1").then((res) =>
    res.json(),
  );

  return <>...</>;
}
```

また、[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)をサポートしているので[tRPC](https://trpc.io/)などなしにデータ変更を実装することが可能です。

```tsx
// app/actions.ts
"use server";

export async function createTodo() {
  // ...
}
```

```tsx
// app/page.tsx
"use client";

import { createTodo } from "./actions";

export default function CreateTodo() {
  return <form action={updateItem}>{/* ... */}</form>;
}
```

App Routerでは可能な限り、**データ取得はServer Components・データ操作はServer Actionsを利用することが推奨**されています。

これはNext.jsの[Patterns and Best Practices](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#fetching-data-on-the-server)でも示されています。

データ取得・データ操作の処理は極力サーバー側に実装することを心がけましょう。

## 例外

### サイト内で利用する外部SaaSのSDK都合

サイト内で利用する外部SaaSのSDK都合で、クライアントサイドでデータ取得や操作を行わないことはあるかもしれません。しかし可能なら、サーバーサイドで処理する方が良いでしょう。
