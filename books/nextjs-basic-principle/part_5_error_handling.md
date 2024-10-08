---
title: "エラーハンドリング"
---

## 要約

App Routerにおけるエラーハンドリングは主に、Server ComponentsとServer Actionsの2つで発生します。

Server Componentsのエラーは、エラー時UIを`error.tsx`や`not-found.tsx`で定義が可能が可能です。一方Server Actionsにおけるエラーは、基本的に戻り値で表現することが推奨されます。

## 背景

Pages Routerにおけるエラーハンドリングは、主に以下の観点で考える必要があります。

- [ページにおけるエラー](#ページにおけるエラー)
- [API Routesにおけるエラー](#api-routesにおけるエラー)

### ページにおけるエラー

Pages Routerにおいてサーバー側エラーが発生すると、エラーページが表示されます。エラーページの定義は、エラーの種類に応じて以下のファイルで定義することが可能です。

- `404.tsx`: 404 Not Found
- `500.tsx`: 500 Internal Server Error

`403 Forbidden`など上記定義以外でエラー時UIをカスタマイズしたい場合には、`_error.tsx`でより高度なエラーハンドリングを行うことが可能です。

### API Routesにおけるエラー

API Routesにおけるエラーハンドリングは、適切なHTTP Status Codeやメッセージの返却が基本となります。API RoutesをGraphQLや[tRPC](https://trpc.io/)などの3rd partyライブラリと統合している場合には、エラーハンドリングの実装がライブラリに依存することもあります。

## 設計・プラクティス

App Routerにおけるエラーハンドリングは大きく分けて、Server ComponentsとServer Actionsの2つで考える必要があります。

### Server Componentsのエラー

App Routerでは、サーバー側エラー時のUIをRoute Segment単位の`error.tsx`で定義することができます。Route Segment単位なのでレイアウトはそのまま、ページ部分だけが`error.tsx`で定義したUIが表示されます。以下は[公式ドキュメント](https://nextjs.org/docs/canary/app/api-reference/file-conventions/error#how-errorjs-works)にある図です。

![エラー時のUIイメージ](/images/nextjs-basic-principle/error-ui.png)

`error.tsx`は主にServer Components、Server Actionsでエラーが発生した場合に利用されます。

:::message
厳密にはSSR時のClient Componentsでエラーが起きた場合にも`error.tsx`が利用されます。
:::

`error.tsx`はpropsで、リロード的振る舞いをする`reset()`を受け取るので、これを利用して再度レンダリングを試みるようなUI実装がよく行われます。

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

また、App Routerでは特別なエラーをthrowするためのAPIとして`notFound()`を提供しています。HTTPにおける404 Not Found相当のUIを通常のエラーと分けたいケースは非常によくあるユースケースで、App Routerではこの際のUIを`not-found.tsx`で定義することが可能です。

https://nextjs.org/docs/canary/app/api-reference/file-conventions/not-found

:::message
多くの場合、`notFound()`はHTTP Status Codeとして404 Not Foundを返しますが、`<Suspens>`内などで利用すると200 OKを返すことがあります。この際、`<meta name="robots" content="noindex" />`タグを挿入してGoogleクローラなどに対してIndexingの必要がないことを示します。
:::

### Server Actionsのエラー

Server Actionsのエラーは、**予想可能なエラー**と**予期せぬエラー**で分けて考える必要があります。

Server Actionsは多くの場合、データ更新の際に呼び出されます。何かしらの理由でデータ更新に失敗したとしても、ユーザーは再度更新をリクエストできることが望ましいUXと考えられます。しかし、Server Actionsではエラーが`throw`されると、前述の通り`error.tsx`で定義したエラー時UIが表示されます。`error.tsx`が表示され、直前までページで入力してた`<form>`の入力内容などが失われると、ユーザーは操作を最初からやり直すことになりかねません。そのため、Server Actionsでエラーは極力`throw`せず、戻り値でエラーを表現することが推奨されます。

以下は[conform](https://ja.conform.guide/integration/nextjs)を使ったServer Actionsにおけるzodバリデーションの実装例です。バリデーションエラー時は`throw`せず、`submission.reply()`を返している点がポイントです。

```tsx
"use server"; // action.ts

import { redirect } from "next/navigation";
import { parseWithZod } from "@conform-to/zod";
import { loginSchema } from "@/app/schema";

export async function login(prevState: unknown, formData: FormData) {
  const submission = parseWithZod(formData, {
    schema: loginSchema,
  });

  if (submission.status !== "success") {
    return submission.reply();
  }

  redirect("/dashboard");
}
```

formライブラリを利用しない場合は、以下のように自身で戻り値を定義しましょう。

```tsx
"use server";

import { redirect } from "next/navigation";

export async function createUser(prevState: any, formData: FormData) {
  const res = await fetch("https://...");
  const json = await res.json();

  if (!res.ok) {
    return { message: "Please enter a valid email" };
  }

  redirect("/dashboard");
}
```

## トレードオフ

### クライアントサイドにおけるレンダリング時エラー

- Client Componentsでのエラーはブラウザ依存など致命的なエラーな可能性があり、リトライでは解決しない可能性が高い
- もしエラーログ収集や特別なUIを出したいなどの場合には、自身で`ErrorBoundary`を実装してハンドリングしましょう

https://ja.react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary
