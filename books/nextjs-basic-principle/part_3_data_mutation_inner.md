---
title: "データ操作とServer Actions"
---

## 背景

Pages Routerではデータ取得のために[getServerSideProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)や[getStaticProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-static-props)が提供されてましたが、データ操作アプローチは公式には提供されていませんでした。そのため、クライアントサイドを主体、または[API Routes](https://nextjs.org/docs/pages/building-your-application/routing/api-routes)を併用した3rd partyライブラリの実装パターンが多く存在します。

- [SWR](https://swr.vercel.app/)
- [React Query](https://react-query.tanstack.com/)
- GraphQL
  - [Apollo Client](https://www.apollographql.com/docs/react/)
  - [Relay](https://relay.dev/)
- [tRPC](https://trpc.io/)
- etc...

しかしAPI RouteはApp Routerにおいて[Route Handler](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)となり、定義の方法や参照できる情報などが変更されました。また、多層のキャッシュを活用しているためデータ操作時にはキャッシュのrevalidate機能との統合が必要になり、これらの実装パターンをそのままApp Routerで利用することは困難です。

## 設計・プラクティス

App Routerにおけるデータ操作は[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)を利用することが推奨されており、これによりtRPCなどなしにデータ変更を実装することが可能です。

```tsx
// app/actions.ts
"use server";

export async function createTodo(formData: FormData) {
  // ...
}
```

```tsx
// app/page.tsx
"use client";

import { createTodo } from "./actions";

export default function CreateTodo() {
  return (
    <form action={createTodo}>
      {/* ... */}
      <button>Create Todo</button>
    </form>
  );
}
```

サーバー側で実行される関数`createTodo`をClient Componentsの`<form>`の`action`propsに直接渡しているのがわかります。実際にsubmit時にはサーバー側で`createTodo`が実行されます。このように非常にシンプルな実装でクライアントサイドからサーバー側関数を呼び出せることにより、開発者はデータ操作の実装に集中できます。

Server Actions自体はReactの仕様ですが、App Router上で実装されているため他にも以下のようなメリットが得られます。

### キャッシュのrevalidate

前述の通りApp Routerでは多層のキャッシュがデフォルトで有効になっており、データ操作時にはキャッシュのrevalidateが必要になります。Server Actions内で[revalidatePath](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)や[revalidateTag](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)を呼び出すと、サーバー側の関連するキャッシュ([Data Cache](https://nextjs.org/docs/app/building-your-application/caching#data-cache)や[Full Route Cache](https://nextjs.org/docs/app/building-your-application/caching#full-route-cache))とクライアントサイドのキャッシュ([Router Cache](https://nextjs.org/docs/app/building-your-application/caching#router-cache))が再検証されます。

```tsx
// app/actions.ts
"use server";

export async function updateTodo() {
  // ...
  revalidateTag("todos");
}
```

### redirect時の通信効率

App Routerではサーバーサイドで呼び出せる[`redirect`](https://nextjs.org/docs/app/building-your-application/routing/redirecting#redirect-function)という関数があります。データ操作後にページをリダレイクトしたいことはよくあるユースケースですが、`redirect`をServer Actions内で呼び出すとレスポンスにリダイレクト先ページの[RSC Payload](https://nextjs.org/docs/app/building-your-application/rendering/server-components#how-are-server-components-rendered)が含まれるため、HTTPリダイレクトをせずに画面遷移できます。これにより、従来データ操作リクエストとリダイレクト後ページ情報のリクエストで2往復は必要だったhttp通信が、1度で済みます。

```tsx
// app/actions.ts
"use server";

import { redirect } from "next/navigation";

export async function createTodo(formData: FormData) {
  console.log("create todo: ", formData.get("title"));

  redirect("/thanks");
}
```

上記のServer Actionsを実際に呼び出すと、遷移先の`/thanks`のRSC Payloadが含まれたレスポンスが返却されます。

```text
2:I[3099,[],""]
3:I[2506,[],""]
0:["lxbJ3SDwnGEl3RnM3bOJ4",[[["",{"children":["thanks",{"children":["__PAGE__",{}]}]},"$undefined","$undefined",true],["",{"children":["thanks",{"children":["__PAGE__",{},[["$L1",[["$","h1",null,{"children":"Thanks page."}],["$","p",null,{"children":"Thank you for submitting!"}]]],null],null]},["$","$L2",null,{"parallelRouterKey":"children","segmentPath":["children","thanks","children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L3",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","styles":null}],null]},[["$","html",null,{"lang":"en","children":["$","body",null,{"children":["$","$L2",null,{"parallelRouterKey":"children","segmentPath":["children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L3",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":[["$","title",null,{"children":"404: This page could not be found."}],["$","div",null,{"style":{"fontFamily":"system-ui,\"Segoe UI\",Roboto,Helvetica,Arial,sans-serif,\"Apple Color Emoji\",\"Segoe UI Emoji\"","height":"100vh","textAlign":"center","display":"flex","flexDirection":"column","alignItems":"center","justifyContent":"center"},"children":["$","div",null,{"children":[["$","style",null,{"dangerouslySetInnerHTML":{"__html":"body{color:#000;background:#fff;margin:0}.next-error-h1{border-right:1px solid rgba(0,0,0,.3)}@media (prefers-color-scheme:dark){body{color:#fff;background:#000}.next-error-h1{border-right:1px solid rgba(255,255,255,.3)}}"}}],["$","h1",null,{"className":"next-error-h1","style":{"display":"inline-block","margin":"0 20px 0 0","padding":"0 23px 0 0","fontSize":24,"fontWeight":500,"verticalAlign":"top","lineHeight":"49px"},"children":"404"}],["$","div",null,{"style":{"display":"inline-block"},"children":["$","h2",null,{"style":{"fontSize":14,"fontWeight":400,"lineHeight":"49px","margin":0},"children":"This page could not be found."}]}]]}]}]],"notFoundStyles":[],"styles":null}]}]}],null],null],[null,"$L4"]]]]
4:[["$","meta","0",{"name":"viewport","content":"width=device-width, initial-scale=1"}],["$","meta","1",{"charSet":"utf-8"}]]
1:null
```

### JavaScript非動作時・未ロード時サポート

App RouterのServer Actionsでは`<form>`の`action`propsにServer Actionsを渡すと、ユーザーがJavaScriptをOFFにしてたり未ロードであっても動作します。

:::message
[公式ドキュメント](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations#behavior)では「Progressive Enhancementのサポート」と称されていますが、厳密にはJavaScript非動作環境のサポートとProgressive Enhancementは異なると筆者は理解しています。詳しくは以下をご参照ください。

https://developer.mozilla.org/ja/docs/Glossary/Progressive_Enhancement

:::

これにより、[FID](https://web.dev/articles/fid?hl=ja)(First Input Delay)の向上も見込めます。実際にはFormライブラリを利用しつつServer Actionsを利用するケースが多いと想定されるので、筆者はJavaScript非動作時もサポートしてるFormライブラリの[Conform](https://conform.guide/)をおすすめします。

https://zenn.dev/akfm/articles/server-actions-with-conform

## 結論

App Routerにおけるデータ操作処理は、Server Actionsで実装することでキャッシュ操作やリダイレクト時の通信効率の向上など多くのメリットを得られます。**データ取得はServer Componentsで、データ操作はServer Actionsで実装する**ことを基本としましょう。

## 例外

### サイト内で利用する外部SaaSのSDK都合

サイト内で利用する外部SaaSのSDK都合でクライアントサイドでデータ取得や操作を行わないことはあるかもしれません。
