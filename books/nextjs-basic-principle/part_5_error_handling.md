---
title: "エラーハンドリング"
---

## 要約

Next.jsにおけるエラーは主に、Server ComponentsとServer Functionsの2つで発生します。

Server Componentsのエラーは、エラー時のUIを`error.tsx`や`not-found.tsx`で定義します。一方Server Functionsにおけるエラーは、基本的に戻り値で表現することが推奨されます。

## 背景

Next.jsにおけるエラーは、クライアントかサーバーか、データ参照か変更かで分けて考える必要があり、具体的には以下の3つを分けて考えることになります。

- Client Components
- Server Components
- Server Functions

特に、Server ComponentsとServer Functionsは外部データに対する操作を伴うことが多いため、エラーが発生する可能性が高くハンドリングが重要になります。

### クライアントサイドにおけるレンダリングエラー

後述のNext.jsが提供するエラーハンドリングは、サーバーサイドで発生したエラーにのみ対応しています。クライアントサイドにおけるレンダリングエラーのハンドリングが必要な場合は、開発者が自身で`<ErrorBoundary>`を定義する必要があります。

https://ja.react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary

また、クライアントサイドにおいてレンダリングエラーが発生する場合、ブラウザの種類やバージョンなどに依存するエラーの可能性が高く、サーバーサイドとは異なりリトライしても回復しない可能性があります。クライアントサイドのエラーは当然ながらサーバーに記録されないので、開発者はエラーが起きた事実さえ把握が難しくなります。

クライアントサイドのエラー実態を把握したい場合、Datadogなどの[RUM↗︎](https://www.datadoghq.com/ja/product/real-user-monitoring/)（Real User Monitoring）導入も同時に検討しましょう。

## 設計・プラクティス

Next.jsにおけるエラーは主に、Server ComponentsとServer Functionsの2つで考える必要があります。

### Server Componentsのエラー

Next.jsでは、Server Componentsの実行中にエラーが発生した時のUIを、Route Segment単位の`error.tsx`で定義することができます。Route Segment単位なのでレイアウトはそのまま、`page.tsx`部分に`error.tsx`で定義したUIが表示されます。以下は[公式ドキュメント↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/error#how-errorjs-works)にある図です。

![エラー時のUIイメージ](/images/nextjs-basic-principle/error-ui.png)

`error.tsx`は主にServer Componentsでエラーが発生した場合に利用されます。

:::message
厳密にはSSR時のClient Componentsでエラーが起きた場合にも`error.tsx`が利用されます。
:::

`error.tsx`はClient Componentsである必要があり、propsで`reset()`を受け取ります。`reset()`は、再度ページのレンダリングを試みるリロード的な振る舞いをします。

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

#### Not Foundエラー

HTTPにおける404 Not FoundエラーはSEO影響もあるため、その他のエラーとは特に区別されることが多いエラーです。Next.jsでは404相当のエラーをthrowするためのAPIとして`notFound()`を提供しており、[`notFound()`↗︎](https://nextjs.org/docs/app/api-reference/functions/not-found)が呼び出された際のUIは[`not-found.tsx`↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/not-found)で定義できます。

:::message
多くの場合`notFound()`を呼び出すとHTTPステータスコードとして404 Not Foundが返されますが、`<Suspense>`内などで`notFound()`を利用すると200 OKが返されることがあります。この際Next.jsは、`<meta name="robots" content="noindex" />`タグを挿入してGoogleクローラなどに対してIndexingの必要がないことを示します。
:::

:::message
Next.jsには[`unauthorized()`↗︎](https://nextjs.org/docs/app/api-reference/functions/unauthorized)や[`forbidden()`↗︎](https://nextjs.org/docs/app/api-reference/functions/forbidden)も提供されていますが、執筆時現在これらは実験的機能となっています。今後変更される可能性もあるので注意しましょう。
:::

### Server Functionsのエラー

Server Functionsのエラーは、**予測可能なエラー**と**予測不能なエラー**で分けて考える必要があります。

Server Functionsは多くの場合、Server Actions^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]としてデータ更新の際に呼び出されます。何かしらの理由でデータ更新に失敗したとしても、ユーザーは再度更新をリクエストできることが望ましいUXと考えられます。しかし、Server Actionsではエラーが`throw`されると、前述の通り`error.tsx`で定義したエラー時のUIが表示されます。`error.tsx`が表示されることで、直前までページで入力していた`<form>`の入力内容などが失われると、ユーザーは操作を最初からやり直すことになりかねません。

そのため、Server Functionsにおける予測可能なエラーは`throw`ではなく、**戻り値でエラーを表現**することが推奨されます。予測不能なエラーに対しては当然ながら対策できないので、予測不能なエラーが発生した場合は`error.tsx`が表示されることは念頭に置いておきましょう。

以下は[conform↗︎](https://ja.conform.guide/integration/nextjs)を使ったServer Actionsにおけるzodバリデーションの実装例です。バリデーションエラー時は`throw`せず、`submission.reply()`を返している点がポイントです。

```tsx
"use server";

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

  // ...

  redirect("/dashboard");
}
```

formライブラリを利用してない場合は、以下のように自身で戻り値を定義しましょう。

```tsx
"use server";

import { redirect } from "next/navigation";

export async function login(prevState: unknown, formData: FormData) {
  const submission = parseWithZod(formData, {
    schema: loginSchema,
  });

  if (formData.get("email") !== "") {
    return { message: "メールアドレスは必須です。" };
  }

  // ...

  redirect("/dashboard");
}
```

## トレードオフ

特になし
