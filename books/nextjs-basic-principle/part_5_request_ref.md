---
title: "リクエスト情報の参照とレスポンス"
---

## 要約

App Routerでは他フレームワークにあるようなリクエストオブジェクト(`req`)やレスポンスオブジェクト(`res`)を参照することはできません。代わりに必要な情報を参照するためのhooksや関数などのAPIが提供されています。

- [useParams](https://nextjs.org/docs/app/api-reference/functions/use-params)
- [useSearchParams](https://nextjs.org/docs/app/api-reference/functions/use-search-params)
- [cookies](https://nextjs.org/docs/app/api-reference/functions/cookies)
- [headers](https://nextjs.org/docs/app/api-reference/functions/headers)
- [notFound](https://nextjs.org/docs/app/api-reference/functions/not-found)
- [permanentRedirect](https://nextjs.org/docs/app/api-reference/functions/permanentRedirect)
- [redirect](https://nextjs.org/docs/app/api-reference/functions/redirect)
- [その他](https://nextjs.org/docs/app/api-reference/functions)

## 背景

Pages Routerはじめ従来のフレームーワークでは、リクエストオブジェクト(`req`)やレスポンスオブジェクト(`res`)を参照することで様々な情報にアクセスしたり、レスポンスをカスタマイズすることができました。

```tsx
export const getServerSideProps = (async ({ req, res }) => {
  // リクエスト情報から`sessionId`というcookie情報を取得
  const sessionId = req.cookies.sessionId;

  // レスポンスヘッダーに`Cache-Control`を設定
  res.setHeader(
    "Cache-Control",
    "public, s-maxage=10, stale-while-revalidate=59",
  );

  // ...

  return { props };
}) satisfies GetServerSideProps<Props>;
```

しかし、App Routerではこれらのオブジェクトを参照することはできません。

## 設計・プラクティス

## トレードオフ
