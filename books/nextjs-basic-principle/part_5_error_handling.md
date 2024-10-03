---
title: "エラーハンドリング"
---

## 要約

App Routerにおけるサーバー側エラー時のUIは`app/not-found.tsx`や各Route Segmentで定義する`error.tsx`で行うことが可能です。また、クライアントサイドのエラーは、従来からある`ErrorBoundary`パターンで対応しましょう。

## 背景

Pages Routerにおけるエラーハンドリングは、主に以下の観点で考える必要があります。

- SSR時のエラー
- API Routesにおけるエラー
- クライアントサイドのエラー

### SSR時のエラー

Pages RouterではSSR時にエラーが発生すると、エラーページが返却されます。エラーページの定義は、エラーの種類に応じて以下のファイルで定義することが可能です。

- `404.tsx`: 404 Not Found
- `500.tsx`: 500 Internal Server Error
- `_error.tsx`: その他のエラー

### API Routesにおけるエラー

API Routesにおけるエラーハンドリングは、適切なHTTP Status Codeやメッセージの返却が基本となります。API Routesを3rd partyライブラリと統合している場合は、エラーハンドリングの実装はそのライブラリに依存することになります。

### クライアントサイドのエラー

クライアントサイドのエラーハンドリングはNext.jsからは提供されていないため、Reactドキュメントなどでも紹介されている`ErrorBoundary`パターンで対応することが可能です。

https://ja.react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary

## 設計・プラクティス

App Routerにおけるエラーハンドリングは、Pages Routerと類似する以下3つの観点で考える必要があります。

- [Server Componentsのエラー](#server-componentsのエラー)
- [Server Actionsのエラー](#server-actionsのエラー)
- [クライアントサイドのエラー](#クライアントサイドのエラー)

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

### Server Actionsのエラー

TBW: 回復可能なエラー、回復不能なエラー

### クライアントサイドのエラー

- Client Componentsでのエラーハンドリングには、従来同様`ErrorBoundary`を利用しましょう

## トレードオフ

特になし
