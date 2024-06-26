---
title: "データ操作はServer Actions"
---

## 背景

Pages Routerではデータ取得のためのAPIは提供されてましたが、データ操作アプローチは公式には提供されていませんでした。そのため、クライアントサイドを主体としたデータ操作の実装パターンが多く存在します。

- [SWR](https://swr.vercel.app/)
- [React Query](https://react-query.tanstack.com/)
- GraphQL
  - [Apollo Client](https://www.apollographql.com/docs/react/)
  - [Relay](https://relay.dev/)
- [tRPC](https://trpc.io/)
- etc...

しかしApp Routerにおいては多層のキャッシュを活用しているため、データ操作時にはキャッシュのrevalidate機能との統合が必要になり、これらの実装パターンをそのまま利用することは困難です。

## プラクティス

App Routerにおけるデータ操作は[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)を利用することが推奨されており、これにより[tRPC](https://trpc.io/)などなしにデータ変更を実装することが可能です。

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

これも前述の[Patterns and Best Practices](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#fetching-data-on-the-server)で、データ操作はServer Actionsを利用することを推奨していることが示されています。

データ操作の処理はServer Actionsを駆使しサーバー側で実装することを心がけましょう。

## 例外

### サイト内で利用する外部SaaSのSDK都合

データ取得と同様に、サイト内で利用する外部SaaSのSDK都合でクライアントサイドでデータ取得や操作を行わないことはあるかもしれません。しかし可能なら、Server Actionsで処理する方が良いでしょう。

todo: リダイレクト時の効率やJSなしで動作する旨追記
