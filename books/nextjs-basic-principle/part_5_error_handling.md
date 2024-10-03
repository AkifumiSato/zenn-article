---
title: "エラーハンドリング"
---

## 要約

App Routerにおけるサーバー側エラー時のUIは`app/not-found.tsx`や各Route Segmentで定義する`error.tsx`で行います。また、クライアントサイドのエラーは、従来からある`ErrorBoundary`パターンで対応しましょう。

## 背景

Pages Routerではサーバー側エラー発生時、エラーページが返却されます。エラーページの定義は、エラーの種類に応じて以下のファイルで定意義可能です。

- 404 Not Found: `404.tsx`
- 500 Internal Server Error: `500.tsx`
- その他のエラー: `_error.tsx`

クライアントサイドのエラーハンドリングは、`ErrorBoundary`と呼ばれるコンポーネントを実装するパターンが広く普及しています。

https://ja.react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary

## 設計・プラクティス

App Routerでも同様にエラー時のUIをファイル規約で定義しますが、Route Segment単位でエラーUIを定義可能になった点が大きな違いです。

- サーバー側エラーは`error.tsx`で定義しましょう
  - Server Actionsのエラーも同様
  - formのバリデーションエラーやトーストで表示したいような回復可能なエラーについては、conformなどのformライブラリを利用すると実装が容易
- クライアントサイドのエラーハンドリングには、従来同様`ErrorBoundary`を定義して利用しましょう

### `error.tsx`

App Routerでは、サーバー側エラー時のUIをRoute Segment単位の`error.tsx`で定義します。Route Segment単位なのでレイアウトはそのままで、ページ部分だけに`error.tsx`で定義したUIが表示されます。

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

### `not-found.tsx`

### Server Actionsの回復可能なエラー

### クライアントサイドのエラーからの回復

- Client Componentsでのエラーハンドリングには、従来同様`ErrorBoundary`を利用しましょう

## トレードオフ

特になし
