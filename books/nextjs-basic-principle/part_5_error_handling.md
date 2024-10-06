---
title: "エラーハンドリング"
---

## 要約

App Routerにおけるエラーハンドリングは主に、Server ComponentsとServer Actionsの2つで発生します。特にServer Actionsについては、回復可能なエラーと回復不能なエラーを区別して実装する必要があります。

Server ComponentsのエラーやServer Actionsにおける回復不能なエラーでは、エラー時UIを`error.tsx`や`not-found.tsx`で定義が可能が可能です。

一方Server Actionsにおける回復可能なエラーは、戻り値でエラーを表現する必要があります。

## 背景

Pages Routerにおけるエラーハンドリングは、主に以下の観点で考える必要があります。

- [ページにおけるエラー](#ページにおけるエラー)
- [API Routesにおけるエラー](#api-routesにおけるエラー)

### ページにおけるエラー

Pages Routerにおいてサーバー側で回復不能なエラーが発生すると、エラーページが表示されます。エラーページの定義は、エラーの種類に応じて以下のファイルで定義することが可能です。

- `404.tsx`: 404 Not Found
- `500.tsx`: 500 Internal Server Error

`403 Forbidden`など上記定義以外でエラー時UIをカスタマイズしたい場合には、`_error.tsx`でより高度なエラーハンドリングを行うことが可能です。

### API Routesにおけるエラー

API Routesにおけるエラーハンドリングは、適切なHTTP Status Codeやメッセージの返却が基本となります。[tRPC](https://trpc.io/)やGraphQLなどを採用し、API Routesを3rd partyライブラリと統合している場合は、エラーハンドリングの実装はそのライブラリに依存することになります。

## 設計・プラクティス

App Routerにおけるエラーハンドリングは、Pages Routerと類似する以下3つの観点で考える必要があります。

- [Server Componentsのエラー](#server-componentsのエラー)
- [Server Actionsのエラー](#server-actionsのエラー)

### Server Componentsのエラー

App Routerでは、サーバー側エラー時のUIをRoute Segment単位の`error.tsx`で定義します。Route Segment単位なのでレイアウトはそのままで、ページ部分だけに`error.tsx`で定義したUIが表示されます。以下は[公式ドキュメント](https://nextjs.org/docs/app/api-reference/file-conventions/error#how-errorjs-works)にある図です。

![エラー時のUIイメージ](/images/nextjs-basic-principle/error-ui.png)

`error.tsx`はServer Components、Server Actions、そしてClient ComponentsのSSR時にエラーが発生した場合に利用されます。`error.tsx`はpropsにリロード的振る舞いをする`reset()`を受け取るので、これを利用して再度レンダリングを試みるようなUI実装がよく行われます。

```tsx
"use client";

import { useEffect } from "react";

export default function ErrorPage({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    console.error(error);
  }, [error]);

  return (
    <div>
      <h2>Something went wrong!</h2>
      <button type="button" onClick={() => reset()}>
        Try again
      </button>
    </div>
  );
}
```

TBW: `not-found.tsx`
TBW: `global-error.tsx`

### Server Actionsのエラー

TBW: 回復可能なエラー、回復不能なエラー

## トレードオフ

### クライアントサイドにおけるレンダリング時エラー

- Client Componentsでのエラーはブラウザ依存など致命的なエラーな可能性があり、リトライでは解決しない可能性が高い
- もしエラーログ収集や特別なUIを出したいなどの場合には、自身で`ErrorBoundary`を実装してハンドリングしましょう

https://ja.react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary
